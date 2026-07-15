# EMQX Auth Path — Decision Memo

**Date:** 2026-07-15
**Status:** Decision pending (user thinking it over)
**Context:** The MQTT transport branch (`feat/mqtt-transport`) was built assuming
**JWT authentication** (spec "Plan A"). Verified against current EMQX docs:
**EMQX Serverless does NOT support JWT auth, HTTP auth, or HTTP authorization** —
those are Dedicated Flex / BYOC only. Serverless supports only **built-in
database** auth (username/password) + **built-in ACL**, both provisionable via
the Serverless management API. So Plan A cannot run on Serverless. This memo lays
out the two viable paths so the decision is made on facts.

Sources: EMQX Cloud docs — [JWT auth](https://docs.emqx.com/en/cloud/latest/deployments/jwt_auth.html),
[HTTP auth](https://docs.emqx.com/en/cloud/latest/deployments/http_auth.html) ("not supported in Serverless"),
[Serverless auth overview](https://docs.emqx.com/en/cloud/latest/deployments/auth_overview.html),
[Publish API](https://docs.emqx.com/en/cloud/latest/api/publish_v5.html),
[Pricing](https://docs.emqx.com/en/cloud/latest/price/pricing.html).

---

## Path A — Serverless + device-key auth (built-in DB) — RECOMMENDED

Stay on the free Serverless tier. Drop JWT. The device authenticates to the
broker with **its existing device key** as the MQTT password; we **provision**
that credential into EMQX's built-in DB at claim time and delete it on unclaim.
Per-device ACL is unnecessary — a single global placeholder rule covers all.

### Why it fits cleanly
`claimDevice` (`lib/documents.ts:47`) already mints the raw device key at claim
time (`generateDeviceKey()`) and hands it to the device via `pendingDeviceKey`
for a one-time fetch. That raw key is exactly what we hand to EMQX as the MQTT
password (EMQX stores its own salted hash). The device already knows its key, so
the config block needs **no secret at all** — just host/port/username.

### Code changes (a small follow-up plan, all still env-gated no-op)
1. **`lib/env.ts`** — remove `MQTT_JWT_SECRET` (unused); keep `EMQX_API_URL`,
   `EMQX_API_KEY`, `EMQX_API_SECRET`, `EMQX_WEBHOOK_SECRET`, `MQTT_BROKER_HOST`,
   `MQTT_BROKER_PORT`.
2. **`lib/mqtt.ts`** —
   - Delete `mintDeviceMqttJwt` + the `jose` dependency use.
   - `buildMqttConfigBlock(deviceId)` → `{ host, port, clientId: deviceId,
     username: deviceId }` (no `password` — firmware uses its stored device key).
   - `mqttEnabled()` drops the `MQTT_JWT_SECRET` check.
   - Add `provisionDeviceMqtt(deviceId, mqttPassword)` and
     `deprovisionDeviceMqtt(deviceId)` — PUT/DELETE against the EMQX Serverless
     authentication-management API (Basic auth with App Key/Secret). Idempotent,
     fail-open (a provisioning hiccup must never fail a device claim; the device
     just falls back to HTTP polling until reprovisioned).
3. **`lib/documents.ts` `claimDevice`** — after minting the key, call
   `provisionDeviceMqtt(deviceId, key)` in both branches (bind-existing and
   create-new), fail-open exactly like the existing `syncDeviceSubscription`
   try/catch at `:72` / `:108`.
4. **Unclaim / revoke / offboard** — call `deprovisionDeviceMqtt(deviceId)` at
   the revert-claim, offboarding return-to-stock, and any hard-delete paths.
5. **EMQX console (one-time)** — one global built-in ACL rule:
   allow `clientid = *` to pub/sub `d/${clientid}/#` (placeholder), so no
   per-device ACL provisioning. Plus the three Data-Integration webhooks
   (ack / heartbeat / presence) exactly as the runbook already documents.
6. **UNCHANGED:** the three webhook routes, `publishCommand`/trigger publish,
   the whole credit/ack path, the config-block wiring. The cross-tenant fix
   (webhook `clientid` scoping) still holds — the ACL placeholder enforces
   topic isolation broker-side, and the webhook scoping enforces it cloud-side.

### Migration note
Already-claimed devices don't have their raw key stored (only the hash), so they
can't be retro-provisioned — they must be re-claimed to get a fresh key that we
provision. The fleet is tiny today (test devices), so re-claim b580 and move on.

### Cost
$0 on the free tier for the current fleet. At large scale, Serverless bills
session-minutes + traffic; an always-on fleet can exceed the free allowance, but
that is a scale-time cost far below Path B's fixed base, and revisitable then.

### Effort
~3–4 task mini-plan (env tweak, mqtt.ts provisioning fns + config change, 2–4
call sites, unit tests for the provisioning fns). I can write and run it.

---

## Path B — Dedicated Flex + JWT (keep the built code)

Switch the broker to EMQX Cloud **Dedicated Flex**, which supports JWT. The
branch code stays essentially as-is.

### Code changes
- Near zero. One verification: EMQX 5's JWT **ACL claim format** — confirm the
  code's `acl: { sub: [...], pub: [...] }` shape matches what the deployment's
  JWT authenticator expects (EMQX 5 also accepts a rule-list form
  `acl: [{ permission, action, topic }]`); adjust `mintDeviceMqttJwt` if needed.
- No provisioning, no claim-path changes, no migration — JWTs are minted on the
  fly in the config response, which is already built.

### Cost
Dedicated Flex is **~$234/month** entry-level (base fee by tier + traffic), per a
third-party pricing source — verify the exact tier price in-console. This is a
fixed monthly cost regardless of fleet size, hard to justify for a handful of
devices pre-scale.

### Effort
Minimal code; the cost is the tradeoff.

---

## Recommendation

**Path A.** It keeps infra at $0 for today's fleet, the extra code is contained
and reuses the existing device-key model (arguably *more* secure than a 30-day
JWT: auth is checked against the live DB on every connect, revocation is
instant), and it leaves ~90% of the shipped branch untouched. Path B only wins
if a $234/mo fixed cost is acceptable to avoid a small, well-scoped code
follow-up — unlikely given the current stage.

Either way, the account-creation steps (Serverless deployment, App Key/Secret,
the three webhooks) are the same starting point; only the auth wiring differs.
