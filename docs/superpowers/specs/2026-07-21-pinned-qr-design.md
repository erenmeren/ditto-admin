# Pinned QR (Sabit QR) — Design Spec

*Date: 2026-07-21 · Status: approved by Eren (conversation), cloud-side scope; firmware leg specced separately later*

## Problem

Tenants may want a QR code displayed **permanently** on a device — e.g. a restaurant
pinning its menu QR to a table device — instead of (or in addition to) the existing
trigger flow where a QR appears only for `qrTimeoutSeconds` after an API call.

## Decisions (locked)

| Question | Decision |
|---|---|
| Scope | **Per device** (`device` row carries its own pinned URL) |
| Billing | **1 credit per set/change**; clearing the pin is free; re-PUT of the identical URL is a no-op and burns no credit |
| Trigger interplay | **Trigger temporarily wins**: triggered QR shows for `qrTimeoutSeconds`, then the device returns to the pinned QR (instead of idle) |
| Control surface | **Both** tenant UI and public API write the same state |
| Credit mechanics | **Immediate debit** on accepted change — NOT reserve→settle→release (see rationale) |
| Firmware | Separate spec/plan; cloud ships first. Old firmware ignores the pin block — behavior stays as today |

## Data model

Two new columns on `device` (no new table):

- `pinnedUrl` — `text`, nullable. `NULL` = no pin.
- `pinnedAt` — `timestamp`, nullable. Set whenever `pinnedUrl` changes to a non-null value.

History/audit: existing `auditLog` gets entries for pin set/change/clear.
Credit ledger: new entry type `pin_change` (amount −1) so billing pages can label it.

## Public API

New scope: **`devices:pin`** (least-privilege; NOT implied by `devices:trigger`).

### `PUT /api/v1/devices/{deviceId}/pin`

- Auth: API key with `devices:pin` scope; device must belong to the key's org.
- Headers: `Idempotency-Key` supported (same mechanism as trigger).
- Body: `{ "url": "https://..." }` — must parse as http(s) URL.
- Semantics:
  - `url` equal to current `pinnedUrl` → no-op: **200 with current pin state,
    no credit burned, no command enqueued** (idempotent).
  - `url` differs → debit 1 credit (`pin_change` ledger entry). Insufficient credits → **402**.
  - On success: update `pinnedUrl`/`pinnedAt`, write audit entry, enqueue delivery
    (see Delivery), return the new pin state.
- Unlike trigger, the device does NOT need to be online — the pin persists and
  applies when the device next connects.

### `DELETE /api/v1/devices/{deviceId}/pin`

- Free. Sets `pinnedUrl = NULL`, audit entry, enqueue delivery. Idempotent
  (deleting a non-existent pin is a no-op success).

## Credit rationale (why immediate debit, not reserve→settle)

Trigger credits use reserve→settle→release because a trigger can fail to display
(device offline/expiry) and must refund. A pin cannot "fail permanently": it is
durable state that is guaranteed to reach the device eventually (config sync on
reconnect/reboot). A hold that expires while a device is offline would release
and make the change free — inconsistent. Immediate debit is simpler and honest.
Implementation: a small `spendCredit(orgId, type)` helper (simple conditional
decrement + ledger row), reusing the ledger table, not the hold machinery.

## Delivery to device (two channels, one truth)

1. **Instant path** — enqueue `deviceCommand` `type: "pin"`, `payload: { url }`
   (`url: null` for clear), delivered over the existing MQTT command channel and
   ack'ed like other commands. The ack does NOT gate the credit.
2. **Durable path** — `/api/device/config` device-specific block gains
   `pin: { url } | null` so a rebooting/reconnecting device recovers the pin.

**ETag caveat (load-bearing):** config `version`/`notModified` is currently
org-level. The per-device pin MUST be folded into the ETag computation (e.g.
hash of org config version + device `pinnedUrl`/`pinnedAt`), otherwise a pin
change is swallowed by the 304 short-circuit.

## Tenant UI

- **Device detail page — "Sabit QR" card**: shows current pin (URL, QR preview,
  pinned-since), actions Set / Change / Remove.
  - Set/Change dialog: URL input + "1 kredi kullanılacak" notice + current credit
    balance; disabled with guidance when balance is 0.
  - RBAC: `canManageTenant` (owner/admin) can write; members see read-only —
    same pattern as pause/resume.
- **Device list**: small 📌 badge on pinned devices.
- **Platform admin device page**: read-only display of pin state.

## Edge cases

- **Paused device**: paused screen overrides the pin (pause halts everything).
- **Device deleted**: pin state goes with the row; nothing extra.
- **Device moved to org pool / another store**: pin is preserved (org-scoped asset).
- **Org runs out of credits**: existing pin keeps displaying (only *changes* cost).
- **Concurrent UI + API writes**: last-writer-wins via `updatedAt`; no locking.
- **Archived org** (`isOrgArchived`): pin endpoints follow the same guards as
  other mutating tenant actions.

## Out of scope (this spec)

- Firmware rendering (pinned screen = existing `qr` screen layout with countdown
  hidden; return-to-pinned instead of return-to-idle) — separate firmware spec.
- Scan analytics for pinned QRs — technically impossible (scan goes phone→URL).
- Per-store or org-wide pin fan-out helpers ("pin all devices") — future.

## Testing

- Unit: URL validation; identical-URL PUT burns no credit; 402 on empty balance;
  `devices:pin` scope enforcement (trigger-scoped key rejected); DELETE idempotency;
  `spendCredit` conditional-decrement race (no negative balance).
- Config: ETag changes when pin changes; 304 preserved when it doesn't.
- UI/action: `canManageTenant` gate on set/clear server actions; member sees
  read-only card.
- Command: pin command enqueued on set and clear; payload shape.
