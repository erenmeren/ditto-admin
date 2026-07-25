# Scoped Pinned QR — device / store / tenant (2026-07-25)

## Problem

Pinned QR (the "always-on" QR a device shows while idle) exists only per
device: `device.pinnedUrl` / `pinnedAt`, set from the tenant device detail
page or `PUT /api/v1/devices/{id}/pin`. Merchants want to pin one URL to a
whole store, or to every device in the tenant, in one action. Today that
means visiting every device page, and a device added later never picks the
URL up.

## Decision summary (approved 2026-07-25)

- **Layered inheritance**, not bulk fan-out: the pin is stored at the level
  it applies to; devices resolve an *effective* pin as
  `device > store > tenant`. A device added to a pinned store inherits
  automatically.
- **Tri-state at device AND store level**: `inherit | custom | none`.
  `none` is an explicit "show no pin here even though a parent has one".
- **Money rule**: a pin change costs **1 credit per device whose effective
  URL actually changes**, charged in one atomic spend; clears/`none` are
  free; identical-URL writes are free no-ops. Inheritance applied by
  membership changes (claim, move, store deletion) is free.
- **New page** `/tenant/pinned-qr` manages all three levels; the device
  detail card grows the tri-state.
- **Public API expands** in the same release: store-level and org-level pin
  endpoints under the existing `devices:pin` scope.
- **No firmware change**: devices only ever see the resolved effective URL
  via the existing `pin` command and `/api/device/config`.

## Non-goals

- No per-schedule or time-windowed pins.
- No admin-panel (platform) pin management — tenant panel + public API only.
- No change to the QR *style* pipeline (org-wide style continues to apply).

## 1. Data model

Migration (one Drizzle migration; strip generated SQL to just these changes —
see the snapshot-drift note in memory):

| Table | New columns |
|---|---|
| `tenant_settings` | `pinned_url text`, `pinned_at timestamp` |
| `store` | `pin_mode text not null default 'inherit'`, `pinned_url text`, `pinned_at timestamp` |
| `device` | `pin_mode text not null default 'inherit'` |

Backfill in the same migration:

```sql
UPDATE device SET pin_mode = 'custom' WHERE pinned_url IS NOT NULL;
```

Existing device pins keep behaving exactly as before.

Invariant (enforced by the write paths, tolerated by the read paths):
`pin_mode = 'custom'` ⇔ `pinned_url IS NOT NULL` at that level. If they ever
disagree (e.g. hand-edited data), resolution treats `custom` with a null URL
as no pin.

TypeScript: `type PinMode = "inherit" | "custom" | "none"` exported from
`lib/pin.ts` alongside the existing `validatePinBody`.

## 2. Effective-pin resolution

New pure module `lib/pin-resolve.ts`:

```ts
resolveEffectivePin(input: {
  device: { pinMode: PinMode; pinnedUrl: string | null; storeId: string | null };
  store: { pinMode: PinMode; pinnedUrl: string | null } | null;  // null = pool device
  tenant: { pinnedUrl: string | null };
}): { url: string | null; source: "device" | "store" | "tenant" | null }
```

Rules, first match wins:

1. `device.pinMode = 'custom'` → device URL, source `device`
2. `device.pinMode = 'none'` → no pin
3. (device inherits) `store` present and `store.pinMode = 'custom'` → store URL, source `store`
4. `store` present and `store.pinMode = 'none'` → no pin
5. otherwise → tenant URL (may be null), source `tenant` when non-null

Pool devices (`storeId = null`) skip 3–4 and inherit the tenant pin
directly.

Consumers: `/api/device/config` (replaces the current
`{ url: device.pinnedUrl ?? null }` at `app/api/device/config/route.ts:22`),
the new page, the device detail card, and the mutation core (to compute
affected devices).

## 3. Money rule and delivery (mutation core)

Extend `lib/pin-service.ts` with scope-aware mutations. The shared core
(used by server actions AND the API routes so the rules exist once):

- `setOrgPin({ url })`, `clearOrgPin()`
- `setStorePin({ storeId, url })`, `setStorePinMode({ storeId, mode })` (for `none`/`inherit`)
- `setDevicePin` / `clearDevicePin` (existing; clear now means → `inherit`,
  and a new `setDevicePinMode` covers `none`/`inherit` explicitly)

For any mutation, the core:

1. Loads the current effective URL for every device that could be affected
   (org change → all org devices; store change → that store's devices;
   device change → that device).
2. Computes the new effective URL per device; **affected** = devices whose
   effective URL differs (including → null).
3. **Paid ⇔ the change sets a URL.** A `PUT {url}` at any scope charges
   `affected × PIN_COST` in **one** `spendCredit` call
   (`action: "pin_change"`; `lib/credits.ts` `spendCredit` already takes an
   arbitrary `cost`) — for a URL set, every affected device's new effective
   URL is that URL, so affected = charged. Insufficient balance rejects the
   whole operation — no partial application. Everything else is free:
   clears/DELETE, mode changes (`none` **and** `inherit`, even when
   switching to `inherit` makes devices pick up a parent pin), same-URL
   no-ops, and membership-driven inheritance. This generalizes today's
   "change costs 1, clear is free".
4. Writes the level's columns (`pinnedUrl`/`pinnedAt`/`pinMode`).
5. Fans out one `deviceCommand` row (`type: "pin"`, payload
   `{ url: <new effective url> }`) + best-effort MQTT publish **per affected
   device** — the existing per-device delivery path, unchanged from the
   device's point of view.
6. Records audit (see §6).

Charge-first ordering and the non-transactional caveat stay as documented in
`lib/pin-service.ts` today (neon-http, no interactive transactions; a crash
between spend and apply fails safe for the business).

### Free inheritance on membership changes

When a device's store membership changes — claim into a store, move between
stores/pool, store deletion moving devices to the pool — the new effective
pin is computed and, if it differs, a free `pin` command is enqueued. No
credits. Hook points: `lib/device-claim.ts`, the device-assignment action(s),
and the store-deletion path.

Known accepted loophole: pin an empty store, then add devices — the devices
inherit free. Accepted because every URL *change* still charges per device;
noted here so it isn't rediscovered as a bug.

## 4. UI

### New page `/tenant/pinned-qr`

Sidebar: "Pinned QR" (Pin icon) in the tenant workspace nav (selected inside
`AppShell` by `workspace`, per the existing pattern). Page follows the shell
rhythm (`PageHeader`, `PageSection`, fragment return — no re-padding).

Sections:

1. **Tenant-wide pin** — card with QR preview rendered in the org QR style
   (`QrSvg` + `qrCornerRadiusPx`/`qrShadowBoxShadow`, same as
   `DevicePinControl`), Set / Change / Remove. Shows how many devices
   currently resolve to it.
2. **Stores** — table: store name, mode badge (Inherit / Custom / None),
   effective URL for that store's inheriting devices, device count. Row
   actions open a dialog to switch mode or set a custom URL.
3. **Device exceptions** — list of devices with `custom` or `none`
   (name, store, mode, URL), each linking to its device detail page.

Every paid confirm dialog states the cost up front:
*"This will update 7 devices — 7 credits (you have 42)."* Submit disabled
when balance is short, mirroring `DevicePinControl`'s current pattern.

### Device detail card

`components/device-pin-control.tsx` becomes tri-state: **Inherit** (default;
shows the inherited URL and its source — "from store pin" / "from tenant
pin" — or "no pin anywhere"), **Custom** (today's behavior), **None**.
Read-only for members; mutations gated by `canManageTenant` exactly as now.

## 5. Public API

Same auth stack as the device pin route (`guardApiRequest`, `devices:pin`
scope, `isOrgArchived` on paid mutations, optional `Idempotency-Key` claimed
before charging). Idempotency keys are namespaced **per endpoint** in the
shared `apiIdempotency` table: existing `pin:` for devices, new `storepin:`
and `orgpin:` — a key reused across endpoints must not replay the wrong
body.

| Route | Semantics |
|---|---|
| `PUT /api/v1/org/pin` `{url}` | set tenant-wide pin |
| `DELETE /api/v1/org/pin` | clear tenant-wide pin (free) |
| `PUT /api/v1/stores/{storeId}/pin` `{url}` or `{mode:"none"|"inherit"}` | set store custom pin / switch mode |
| `DELETE /api/v1/stores/{storeId}/pin` | store → `inherit` (free) |
| `PUT /api/v1/devices/{deviceId}/pin` `{url}` or `{mode:"none"|"inherit"}` | set device custom pin / switch mode |
| `DELETE /api/v1/devices/{deviceId}/pin` | device → `inherit` (free) |

Responses include the written state plus `affectedDevices` and
`creditsCharged`. Insufficient balance → `402 insufficient_credits` with the
required amount in the message.

**Intentional behavior change**: `DELETE /api/v1/devices/{id}/pin` now sets
`inherit` (fall back to store/tenant pin) instead of guaranteeing "no pin".
Callers that want the device blank regardless of parents use
`PUT {mode:"none"}`. With no store/tenant pin configured the observable
behavior is identical to today. Document in the API docs/changelog.

## 6. Audit

New `AUDIT` actions in `lib/audit.ts`: `orgPinSet` / `orgPinCleared`,
`storePinSet` / `storePinCleared`, `devicePinModeChanged`; existing
`devicePinSet` / `devicePinCleared` stay. Metadata carries `via`
(`ui`/`api`), `url` where applicable, `affectedDevices`, `creditsCharged`.
Friendly labels added to the audit-label map for the tenant Activity table.

## 7. Testing

- **Unit**: `resolveEffectivePin` truth table (all mode combinations, pool
  devices, custom-with-null-URL tolerance); affected/charged counting for
  org/store/device changes including mixed outcomes (some devices gain,
  some keep, some had overrides).
- **Routes**: new org/store pin route tests mirroring the existing device
  pin route suite — scope rejection, 402 on short balance, idempotency
  claim/replay/release, archived-org gating, namespacing (a `pin:` key must
  not replay a `storepin:` body).
- **Actions**: RBAC (member rejected), revalidation paths, cost preview
  correctness.
- **Config route**: serves the resolved effective URL for each mode
  combination.

## 8. Edge cases

- **Empty store / zero affected**: 0 credits, columns still written — the
  pin waits for future devices (see accepted loophole, §3).
- **Store deletion → pool**: devices may switch from store pin to tenant
  pin; free pin commands enqueued (§3).
- **Archived org**: paid mutations blocked (`isOrgArchived`), clears and
  `none` allowed — matches the existing device pin route.
- **Offline devices**: unchanged — `pin` commands have no `expiresAt`; the
  device converges via command poll or config fetch on reconnect.
- **Concurrent scoped changes**: last write wins per level; per-device
  delivery remains ordered by command creation. No new locking — consistent
  with the codebase's non-transactional style.

## Out of scope / later

- Scheduled or rotating pins.
- Platform-admin visibility into tenant pins (could join the admin device
  detail page later).
- Per-device pin via the new page (the device detail card already covers
  it; the exceptions list links there).
