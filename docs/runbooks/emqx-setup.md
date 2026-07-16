# EMQX Cloud Serverless — Setup Runbook

One-time setup to activate the MQTT device transport. Until these steps are
done and the env vars are set, the cloud runs in HTTP-polling mode (no-op).

## 1. Create the deployment
- EMQX Cloud → Serverless → region **eu-central-1 (Frankfurt)**.
- Note the connection host (`xxxx.eu-central-1.emqxsl.com`) → `MQTT_BROKER_HOST`.
- TLS listener is on port **8883** → `MQTT_BROKER_PORT`.

## 2. API key (for HTTP publish)
- Console → API Keys → create. → `EMQX_API_KEY` / `EMQX_API_SECRET`.
- The HTTP API base (`https://<host>:8443/api/v5`) → `EMQX_API_URL`.

## 3. Authentication — built-in database (Serverless does NOT support JWT)
- **The built-in-database password authenticator is PRE-CREATED on Serverless** —
  no console step needed. Verify: `GET {EMQX_API_URL}/authentication/password_based%3Abuilt_in_database/users`
  returns `200` with `{"data":[],...}`.
- **IMPORTANT — encode the colon.** Serverless is a namespaced/multi-tenant
  deployment: the API key can `publish` and manage this authenticator's users,
  but the path's authenticator id colon MUST be percent-encoded as `%3A`. The
  literal-colon path (`password_based:built_in_database`) returns **403**; the
  encoded path (`password_based%3Abuilt_in_database`) returns 200. (This is why
  `emqxAuthUsersBase()` in `lib/mqtt.ts` uses `%3A`.)
- Credentials are provisioned automatically by the app at device-claim time
  (`provisionDeviceMqtt` → `POST .../authentication/password_based%3Abuilt_in_database/users`,
  `{user_id: deviceId, password: <device key>, is_superuser: false}` → `201`;
  a duplicate `409` → `PUT .../users/{deviceId}` updates the password), and
  deleted on device delete/unclaim (`DELETE .../users/{deviceId}` → `204`, `404`
  treated as already-gone). **The device's MQTT password is its device key**
  (the same key it uses for HTTP `Authorization: Bearer`), username = deviceId.
  Verified live against the deployment API (201/409/204/404 all behave).

## 3b. Authorization — one global `${username}` ACL rule (set via API)
The admin authz endpoints (`/authorization/sources`, `/authorization/settings`)
return **403** for the namespaced Serverless key, but the built-in-DB **all-rules**
endpoint is accessible. Set one global rule set with a **catch-all deny** so
isolation holds regardless of the (inaccessible) no-match default:

```bash
curl -X POST -u "$EMQX_API_KEY:$EMQX_API_SECRET" \
  -H 'Content-Type: application/json' \
  "$EMQX_API_URL/authorization/sources/built_in_database/rules/all" \
  -d '{"rules":[
    {"topic":"d/${username}/cmd","permission":"allow","action":"subscribe"},
    {"topic":"d/${username}/ack","permission":"allow","action":"publish"},
    {"topic":"d/${username}/hb","permission":"allow","action":"publish"},
    {"topic":"#","permission":"deny","action":"all"}
  ]}'   # → 204
```

- Use `${username}`, NOT `${clientid}`. `${username}` is the authenticated
  identity (verified against the built-in-DB password = device key at connect).
  `${clientid}` is client-supplied and NOT verified against the username by
  default, so keying isolation on it would be spoofable. Since username =
  deviceId, this one rule isolates every device — no per-device ACL needed.
- The trailing `deny #` makes a topic that matches none of the allow rules
  denied, so a device can only ever reach `d/<its own username>/…`.
- Verify: `GET {EMQX_API_URL}/authorization/sources/built_in_database/rules/all`
  shows the four rules. Confirmed applied on the live deployment (2026-07-16).
- **HIL validation:** run the spoof test in step 6 (valid credential, mismatched
  `clientid`) to confirm isolation genuinely keys on `username`, not `clientid`.

## 4. Data-Integration webhooks (broker → cloud)
Create three HTTP-action webhooks, each sending header
`x-emqx-webhook-secret: <EMQX_WEBHOOK_SECRET>`:
- **ack:** rule `SELECT payload, username FROM "d/+/ack"` → POST `<APP_URL>/api/mqtt/ack`,
  body = `{...payload, "clientid": username}` (order matters — the broker-injected
  `username` must be spread LAST so it wins over any `clientid` a device might put
  in its own payload; the route trusts the `clientid` field for device-ownership
  scoping, so a device-controlled override would defeat that check). The field is
  mandatory — the route scopes the ack update to that device's own commands, and
  rejects the request with 400 if it's missing.
- **heartbeat:** rule `SELECT payload, username FROM "d/+/hb"` → POST
  `<APP_URL>/api/mqtt/heartbeat`, body = `{...payload, "clientid": username}`.
- **presence:** events `client.connected`, `client.disconnected` → POST
  `<APP_URL>/api/mqtt/presence`, body includes `event` and the event's `username`
  as `clientid` (these events expose both `clientid` and `username`; use `username`).

**The identity field the routes read is still named `clientid` for route
compatibility, but its VALUE must always be the authenticated `username`, never
the connection `clientid`.** The whole device-ownership/credit-settlement model
(ack scoping, heartbeat status, presence online/offline) trusts this field —
`clientid` is client-supplied and unverified, so populating it with the raw
`clientid` would let a device holding one valid credential spoof another
device's identity by opening a connection with a different `clientid`.

## 5. Set env vars
Set the following required env vars in Vercel (prod) and `.env.local` (local),
then redeploy:
- `EMQX_API_URL` — EMQX HTTP API base (`https://<host>:8443/api/v5`)
- `EMQX_API_KEY` / `EMQX_API_SECRET` — management credentials
- `EMQX_WEBHOOK_SECRET` — shared header secret for inbound webhooks
- `MQTT_BROKER_HOST` — connection host (`xxxx.eu-central-1.emqxsl.com`)
- `MQTT_BROKER_PORT` — TLS listener port (`8883`)

Validate with the desk device (b580): trigger via the public API and confirm
the QR renders in < 1 s, then kill the broker connection and confirm HTTP
polling resumes.

## 6. Spoof test (identity keyed on `username`, not `clientid`)
Connect to the broker with device A's valid credential (`username=<deviceA
id>`, `password=<deviceA key>`) but set the MQTT `clientid` to device B's id.
Confirm:
- The connection CANNOT subscribe to `d/<deviceB>/cmd` — ACL is evaluated on
  `${username}` (device A), so it's still confined to `d/<deviceA>/cmd`.
- Publishing an ack/heartbeat on this connection settles/attributes against
  device A's commands and status, never device B's — the webhook payload
  carries the connection's authenticated `username` (device A) as `clientid`,
  regardless of what `clientid` device A's client claimed.
This proves the whole isolation + attribution model survives a spoofed
`clientid` from a device that only holds one valid credential.

## Notes
- Device credentials (username = deviceId, password = device key) are
  provisioned at claim time and persist until device deletion or unclaim.
  No TTL, no re-issue cycle: the broker trusts the built-in DB and the app
  trusts the device's device key (same used for HTTP auth) — both sides in sync.
- If a device's credential is somehow compromised, delete and re-claim the
  device to rotate its key; the old entry in EMQX will be cleaned up via
  the management API.
