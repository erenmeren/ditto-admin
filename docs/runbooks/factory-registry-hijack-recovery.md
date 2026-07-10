# Factory Registry Hijack-Recovery Runbook

_Owner: platform team · Last reviewed: 2026-07-10_

## The exposure this covers

Zero-touch provisioning (see `docs/superpowers/specs/2026-07-09-factory-registry-provisioning-design.md`)
lets a pre-allocated device claim its key with **no human approval**: any poll
of `GET /api/device/claim` carrying an `allocated` serial and *any* well-formed
pairing code auto-claims that serial's org — see `shouldAutoClaim` /
`autoClaimDevice` in `lib/factory-registry.ts`.

The serial is printed on the box and is **not a secret** — it was never meant
to authenticate anything, only to match inventory. So during the window
between "serial allocated to a customer" and "the real device claims it," a
bare serial plus a guessed/observed pairing code is enough to mint that org's
device key instead of the legitimate installer's device.

**This is an accepted trade-off, not a bug.** The alternative is dropping
zero-touch install entirely. The deterrents that keep the window narrow and
the blast radius small:

- **One-shot transition.** Auto-claim only fires on `allocated → claimed`. Once
  a serial is `claimed` (legitimately or by a hijacker), it never re-mints a
  key — a second claim attempt on the same serial is always rejected. The
  window closes the instant *either* party claims first.
- **Rate limits.** `GET /api/device/claim` is rate-limited per-code (30/min)
  and per-IP (60/min) — see `app/api/device/claim/route.ts` — so a hijacker
  can't brute-force the pairing-code space quickly.
- **Audit trail.** Every auto-claim writes a `device.auto_claimed` audit event
  (`lib/audit.ts`, `AUDIT.deviceAutoClaimed`) scoped to the claimed org, and
  fires a best-effort platform-admin email (`Device auto-claimed: <serial>`,
  see `lib/registry-emails.ts` + the `after()` callback in
  `app/api/device/claim/route.ts`) with the serial, org, device id, and
  timestamp.

## How to detect a hijack

Any of these is a signal worth investigating:

1. **A `device.auto_claimed` admin email you don't recognize** — an org
   getting an auto-claim notification for a serial that isn't currently being
   installed for them.
2. **The org's audit log** shows `device.auto_claimed` at a time no installer
   was on site (tenant Activity page, or query `audit_log` directly for
   `action = 'device.auto_claimed'`).
3. **The legitimate installer reports** the device never claims / shows
   `{status: "pending"}` forever, or the admin inventory page shows the serial
   already `claimed` before the real device ever polled.
4. **Admin inventory badges** — `serialConflict` (the real device's later
   stamp attempt hit a serial already taken — see `stampDeviceSerial` in
   `lib/factory-registry.ts`) or an `unregistered` serial appearing where an
   allocated one was expected are secondary signs something claimed out of
   band.

## Recovery (3 steps)

### 1. Delete the rogue device row

Admin → Devices → the auto-claimed device's detail page → **Delete device**.
This removes the hijacker's device row (and its key hash — the key stops
working immediately) without touching the org's other devices.

If you need to confirm which row is rogue first, cross-check the device id in
the admin-alert email against the org's device list — the legitimate device
(if it has since polled) will show as `serialConflict` rather than as a
normal claimed device.

### 2. Revert the registry row to `allocated`

Admin → Inventory → find the serial → **Revert claim…** (only shown on
`claimed` rows). The action (`revertRegistryClaim` in
`lib/factory-registry.ts`, via `revertRegistryClaimAction`) refuses with
"Delete the linked device first." if a `deviceId` is still linked — so step 1
above must actually be done before this will work — and on success it writes
a `registry.claim_reverted` audit event (`AUDIT.registryClaimReverted`,
`lib/audit.ts`) scoped to the allocated org, when the row has one.

Verify first that `allocated_organization_id` / `allocated_store_id` on that
row are still the correct customer/store (the revert action only clears
`status` and `claimed_at`, never those allocation columns — that's what
re-arms the pending install) — if the wrong org/store is allocated, fix that
in the admin Inventory page's allocate flow instead.

<details>
<summary>Fallback: direct SQL (only if the UI action is unavailable)</summary>

```sql
update factory_device
set status = 'allocated',
    claimed_at = null
where serial = '<serial>'
  and status = 'claimed'
  and device_id is null;
```

</details>

### 3. Re-run the legitimate install

Have the installer power-cycle / re-trigger provisioning on the real device.
It will poll `GET /api/device/claim` again, find the serial back in
`allocated`, and auto-claim normally — this mints a **fresh** device key (the
old key, tied to the deleted rogue row, is already gone from step 1).

## After recovery

- Confirm the new `device.auto_claimed` audit event / admin email corresponds
  to the real device (right timestamp, right installer-reported serial).
- If the org was billed for credits consumed by the rogue device before it was
  deleted, that's a separate billing-support decision — this runbook only
  covers registry/device state.
- If hijacks recur on the same batch/serial range, treat it as a signal to
  investigate how the pairing code or serial leaked (e.g. box labels visible
  before install, a compromised installer, or a fulfillment-partner leak) —
  the deterrents above slow an opportunistic hijack, they don't stop a
  targeted one with code+serial in hand.
