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

## 3. JWT authentication
- Console → Access Control → Authentication → add **JWT**.
- Algorithm **HS256**, secret = `MQTT_JWT_SECRET` (`openssl rand -base64 32`).
- Enable ACL-from-JWT so the `acl.{sub,pub}` claims are enforced.
- **Verify EMQX Serverless supports JWT auth + ACL claims on your plan.**
  If not, fall back to per-device username/password provisioned via the
  built-in-database authenticator at device-claim time (Plan B in the spec) —
  this changes `mintDeviceMqttJwt`/`buildMqttConfigBlock` and the claim flow.

## 4. Data-Integration webhooks (broker → cloud)
Create three HTTP-action webhooks, each sending header
`x-emqx-webhook-secret: <EMQX_WEBHOOK_SECRET>`:
- **ack:** rule `SELECT payload, clientid FROM "d/+/ack"` → POST `<APP_URL>/api/mqtt/ack`,
  body = `{"clientid": clientid, ...payload}`. The `clientid` is mandatory — the
  route scopes the ack update to that device's own commands, and rejects the
  request with 400 if it's missing.
- **heartbeat:** rule `SELECT payload, clientid FROM "d/+/hb"` → POST
  `<APP_URL>/api/mqtt/heartbeat`, body = `{"clientid": clientid, ...payload}`.
- **presence:** events `client.connected`, `client.disconnected` → POST
  `<APP_URL>/api/mqtt/presence`, body includes `event` and `clientid`.

## 5. Set env vars
Set all `EMQX_*` / `MQTT_*` vars in Vercel (prod) and `.env.local` (local),
then redeploy. Validate with the desk device (b580): trigger via the public
API and confirm the QR renders in < 1 s, then kill the broker connection and
confirm HTTP polling resumes.

## Notes
- The per-device MQTT JWT has a 30-day TTL and is only re-issued on a full 200
  `/api/device/config` response — a 304 (not-modified) does NOT refresh it.
  Firmware must perform a periodic full fetch (without `If-None-Match`) well
  inside the 30-day window to pick up a fresh JWT, or the broker connection
  will start rejecting it and the device will silently fall back to HTTP
  polling.
