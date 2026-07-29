# MQTT-only device transport — design

**Date:** 2026-07-29
**Status:** Approved (design)
**Supersedes the transport half of:** `2026-07-15-mqtt-transport-design.md` (which kept HTTP polling as a permanent fallback)

## Goal

Every recurring device↔cloud exchange runs over MQTT. The HTTP device API is
deleted, not deprecated. There is deliberately **no fallback**: if the broker is
unreachable, the fleet is silent rather than slow.

Two exchanges stay on HTTPS because MQTT cannot carry them:

- **Bootstrap claim** (`GET /api/device/claim`) — the device has no device key
  yet, so it cannot authenticate to the broker. Happens once per device lifetime.
- **Binary downloads** — the OTA `.bin` (~2 MB) and R2 branding assets, fetched
  from short-lived presigned URLs by `esp_https_ota` / `cloud_fetch_asset`.

Everything else — commands, acks, heartbeat, presence, config, OTA manifest —
is MQTT. **The device never polls.** It publishes its state and asks for config
at boot; the cloud decides and pushes.

## Channel contract

Existing topics (unchanged): `d/{id}/cmd` (down), `d/{id}/ack`, `d/{id}/hb`
(up), plus EMQX presence events.

**One new topic:** `d/{id}/cfg/get` (up) — the device's "send me my config"
request.

Config and the OTA manifest travel **on the existing `d/{id}/cmd` topic** as new
command types (`type: "config"`, `type: "ota"`). The device already subscribes to
that topic and `dispatch_command` is already transport-agnostic, so the firmware
needs two new handlers — no new subscription, no new parser scaffolding.

### Flows

| Trigger | Flow |
|---|---|
| Boot / reconnect | Device publishes `cfg/get` → EMQX rule → `POST /api/mqtt/config-request` → cloud builds config **at that moment** (fresh presigns) and publishes `type:"config"` on `cmd` |
| Config / branding / device-settings change | `enqueueConfigChangedForOrg` publishes the **config itself**, not a nudge, to every claimed device in that org. Offline devices miss it and pick it up at boot |
| Firmware published | `publish-firmware` publishes `type:"ota"` with `{version, url, sha256, size}` to every claimed device across all orgs (firmware releases are platform-wide); the device downloads the `.bin` over presigned HTTPS |
| Device is behind | The heartbeat payload already carries `version`; the hb webhook feeds it to the existing `firmwareUpdateAvailable` helper (`lib/device-status.ts:36` — "differs from latest", not semver ordering) and publishes `type:"ota"` when it returns true. This is how an OTA missed while powered off gets caught — no extra topic |

Both new command types are **persisted as `deviceCommand` rows** (`type` `"config"`
/ `"ota"`, `billing: "included"`), like every other command. That keeps the
delivered/acked observability the admin UI already renders — "did this device
actually receive its config?" is worth answering — and lets hb-republish recover
them. One row per delivery.

### Why the config-changed nudge is removed

A "config changed, go fetch it" message only made sense when the device had an
HTTP GET to fall back on. With HTTP gone, the message must **carry** the config.
Sending it directly saves a round trip and a `deviceCommand` row.

### Presign freshness

The config payload embeds 300-second presigned R2 URLs (`lib/storage.ts`
`presignedGetUrl` default). Because config is generated per request / per change
and published immediately, the device receives URLs seconds old. Retained
messages are deliberately **not** used — a retained copy would serve dead URLs
after six minutes.

### Cost, stated plainly

`cfg/get` needs a **fifth EMQX console rule**. That widens exposure to the
"choose HTTP Server, not Republish" trap that has broken this system twice. R2
below is the mitigation.

## What gets deleted

### Cloud

| File | Reason |
|---|---|
| `app/api/device/commands/route.ts` | the polling itself |
| `app/api/device/commands/ack/route.ts` | acks arrive via `/api/mqtt/ack` (already live) |
| `app/api/device/config/route.ts` | config is pushed on `cmd` |
| `app/api/device/firmware/route.ts` | the manifest is pushed on `cmd` |
| `lib/device-auth.ts` | those four routes were its only callers → dead code. The device key remains, purely as the MQTT password |

`app/api/device/claim/route.ts` stays (unauthenticated bootstrap).

### Firmware

Deleted: `cloud_get_commands`, `cloud_get_commands_to`, `cloud_ack_command`,
`commands_handle_body`, `cloud_get_config`, `cloud_get_firmware`,
`cloud_post_document` (the last one is already dead — ingest was removed in the
trigger-only pivot). All `mqtt_up` branching in `app_state.c` and `POLL_IDLE_MS`
go with them, leaving a single path.

Kept on HTTPS: `cloud_claim_poll`, `cloud_fetch_asset`, `esp_https_ota`.

## What gets added

### Cloud

- `app/api/mqtt/config-request/route.ts` — the `cfg/get` webhook. Identity comes
  from the **authenticated `username`**, never `clientid` (client-supplied and
  spoofable); auth is the shared `x-emqx-webhook-secret`, matching the other
  three webhooks.
- `lib/mqtt-config-push.ts` — one seam: resolve effective pin → presign → publish
  `type:"config"`. Called by `enqueueConfigChangedForOrg`, by the
  `config-request` webhook, and by nothing else.
- `lib/mqtt-ota-push.ts` — publish the manifest. Called by `publish-firmware` and
  by the hb version comparison.
- `mqttWebhookPing` table, upserted by all five webhooks (the four existing ones
  gain a one-line write), plus the "MQTT health" card in admin that reads it — see R2.

### Firmware

- `mqtt_publish_cfg_get()`, called at boot/reconnect after a 0–3 s random jitter.
- A `type:"config"` handler and a `type:"ota"` handler on the existing dispatch path.

### Refactors carried by this work

The JSON→`device_config_t` parsing inside `cloud_get_config` is valuable and must
survive the transport change; it is extracted as
`cloud_config_parse_body(const char *, device_config_t *)` and called by the MQTT
handler. Likewise `ota.c:19`'s `cloud_get_firmware(&m)` becomes
`ota_start_with_manifest(&m)` — the download, verification and reboot code is
untouched.

## Behavior change: a trigger can no longer vanish silently

`app/api/v1/devices/[deviceId]/trigger/route.ts` publishes best-effort today; its
comment says the device will get the command "via HTTP polling and/or the
heartbeat republish". That safety net is being removed.

New behavior: **publish fails → mark the command `failed`, cancel the credit
reservation, return 503.** Two reasons:

1. A trigger delivered five minutes later is worthless — the customer is standing
   at the counter now.
2. Leaving the row `pending` means hb-republish delivers it later and the device
   shows an unwanted QR.

This applies to `trigger` only. `config` and `ota` are desired-state messages;
late delivery is the correct behavior for them and they may stay `pending`.

## Behavior that does not change

- **Online status / `lastSeenAt`** now come only from the hb (5 min) and presence
  (instant, via LWT). `OFFLINE_MINUTES` is 15 (`lib/device-status.ts:4`), so three
  missed heartbeats are tolerated. No threshold change needed.
- **hb republish stays load-bearing.** A QoS-1 publish to a device with no live
  subscription is dropped by the broker; the existing "republish stale pending
  commands on heartbeat, capped at 15 minutes" logic is now the only recovery
  path for desired-state messages.
- **When MQTT is down** the device relies on esp-mqtt's own backoff reconnect and
  stays on the idle screen. No broker-specific error screen is added — the Wi-Fi
  widget already shows network state, and a broker outage is in practice a Wi-Fi
  outage.

## Risks

**R1 — EMQX is a single point of failure.** Accepted as the explicit cost of "no
second option": during an outage there are no triggers, no config, no OTA. The
only mitigation is visibility — publish failures now surface as a 503 to the API
caller and are captured by Sentry (already wired). No new alert infrastructure.

**R2 — Webhook fragility across five channels.** The "action must be HTTP Server,
not Republish" trap has bitten twice, and the second time **credit-settle broke
silently**. Removing the HTTP safety net makes that failure class more expensive.
Mitigation: a small `mqttWebhookPing` table (channel PK, `lastAt`,
`lastDeviceId` — five rows total) upserted by each webhook, surfaced as an "MQTT
health" card in admin showing each channel's last-heard time. One silent channel
next to four talking ones makes a misconfiguration obvious at a glance. EMQX's
rules API returns 403 for namespaced keys, so configuration cannot be verified
from code — diagnosis has to come from traffic. Cost: one migration.

**R3 — Deploy ordering.** Reversing this strands the device on USB recovery.

| Phase | Work | Why in this position |
|---|---|---|
| **A — cloud, additive only** | `cfg/get` webhook, config/ota push seams, `mqttWebhookPing`, fifth EMQX rule. **The four old routes stay.** Deploy to prod | Fully reversible; existing firmware is unaffected |
| **B — firmware** | Delete the HTTP calls, add the `config`/`ota` handlers → 0.18.0 → USB flash + HIL | The new firmware asks for config over MQTT, so it cannot work until Phase A is in prod |
| **B2 — publish OTA + confirm convergence** | Publish 0.18.0, verify `b580` actually took it | — |
| **C — cloud, subtractive** | Delete the four routes and `lib/device-auth.ts`, enable the trigger hard-fail, update tests | Doing C before B2 is confirmed cuts the device's config **and** OTA paths at once — the only way back is USB |

**R4 — Two leftover device rows.** The `0.6.0-m6b` rows (one never claimed)
cannot speak MQTT and are permanently dead after Phase C. Delete them in Phase C.
The real fleet is a single device (`b580`, 0.17.1), which is precisely why now is
the cheapest moment to make this change.

**R5 — Boot storm.** If a store's devices all power up at once, their
simultaneous `cfg/get` requests produce a burst of presigning. Negligible at
today's fleet size, but the insurance is one line: the 0–3 s jitter before
`cfg/get` specified above.

## Testing

Deleted route tests go away. Added:

- `config-request` webhook: rejects a wrong secret, scopes identity to the
  authenticated `username`, rejects a cross-tenant spoof attempt.
- config-push seam: resolves the effective pin correctly and presigns freshly on
  every call.
- ota-push: version-comparison boundaries.
- trigger publish failure: command marked `failed`, hold cancelled, 503 returned.
- Firmware cfg-harness: `type:"config"` and `type:"ota"` parse tests.

Gates: `npm test` (438 passing today), `tsc`, `next build`; firmware `make test`
and `idf.py build`.

## Documentation to update

- `docs/runbooks/emqx-setup.md` — the fifth rule, with the Republish warning.
- `CLAUDE.md` — the "Device trigger flow" section describes poll + ack and will be
  wrong; rewrite it around the MQTT channel contract.
