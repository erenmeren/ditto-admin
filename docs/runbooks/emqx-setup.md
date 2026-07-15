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
- Console → Access Control → Authentication → add **Password-Based → Built-in Database**.
- Password hashing: default (bcrypt/sha256) is fine — the cloud provisions
  credentials via the management API; nothing to configure per-device here.
- Credentials are provisioned automatically by the app at device-claim time
  (`provisionDeviceMqtt` → `POST /api/v5/authentication/password_based:built_in_database/users`,
  `{user_id: deviceId, password: <device key>, is_superuser: false}`), and deleted
  on device delete/unclaim. **The device's MQTT password is its device key**
  (the same key it uses for HTTP `Authorization: Bearer`), username = deviceId.

## 3b. Authorization — one global ACL rule with a placeholder
- Console → Access Control → Authorization → **Built-in Database**.
- Add a single rule that confines every device to its own topics using the
  `${clientid}` placeholder:
  - allow **subscribe** to `d/${clientid}/cmd`
  - allow **publish** to `d/${clientid}/ack` and `d/${clientid}/hb`
  - (a catch-all deny is the default)
- Because username = clientId = deviceId, this one rule isolates every device
  with no per-device ACL provisioning.
- **Validation:** after setup, confirm device A cannot subscribe to
  `d/<deviceB>/cmd` (a leaked credential must not reach another device's topics).
  If the Serverless tier does not honor `${clientid}` placeholders in built-in
  authorization, fall back to per-device ACL provisioning via the management API
  in `provisionDeviceMqtt` (contingency — not built by default).

## 4. Data-Integration webhooks (broker → cloud)
Create three HTTP-action webhooks, each sending header
`x-emqx-webhook-secret: <EMQX_WEBHOOK_SECRET>`:
- **ack:** rule `SELECT payload, clientid FROM "d/+/ack"` → POST `<APP_URL>/api/mqtt/ack`,
  body = `{...payload, "clientid": clientid}` (order matters — the broker-injected
  `clientid` must be spread LAST so it wins over any `clientid` a device might put
  in its own payload; the route trusts `clientid` for device-ownership scoping,
  so a device-controlled override would defeat that check). The `clientid` is
  mandatory — the route scopes the ack update to that device's own commands, and
  rejects the request with 400 if it's missing.
- **heartbeat:** rule `SELECT payload, clientid FROM "d/+/hb"` → POST
  `<APP_URL>/api/mqtt/heartbeat`, body = `{...payload, "clientid": clientid}`.
- **presence:** events `client.connected`, `client.disconnected` → POST
  `<APP_URL>/api/mqtt/presence`, body includes `event` and `clientid`.

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

## Notes
- Device credentials (username = deviceId, password = device key) are
  provisioned at claim time and persist until device deletion or unclaim.
  No TTL, no re-issue cycle: the broker trusts the built-in DB and the app
  trusts the device's device key (same used for HTTP auth) — both sides in sync.
- If a device's credential is somehow compromised, delete and re-claim the
  device to rotate its key; the old entry in EMQX will be cleaned up via
  the management API.
