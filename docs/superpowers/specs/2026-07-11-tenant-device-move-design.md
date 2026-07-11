# Tenant Device Move Control — Design

**Date:** 2026-07-11
**Scope:** UI-only follow-up to the store-deletion/unassigned-pool feature (shipped 9152104/04d3165). The tenant-scoped `assignDeviceToStore(deviceId, storeId)` action (owner/admin enforced server-side, audits `deviceReassigned`, org-scoped UPDATE) already supports moving a device between live stores; only the tenant UI surface is missing.

## Decision (brainstormed)

Placement: the tenant device detail page (`/tenant/stores/[storeId]/[deviceId]`) — a "Move to store" control next to the existing pause control. Rejected: device-card menus on the store page (introduces a new menu pattern on cards); both (YAGNI for v1).

## Design

- **`components/device-move-control.tsx`** (new, client): mirrors the `DevicePauseControl`/`StoreDeleteButton` idiom — outline "Move to store" button opening a shadcn `Dialog` with a target-store `Select` (choices passed in; Move disabled until picked), `assignDeviceToStore(deviceId, targetId)` on confirm, sonner toast, then `router.push(\`/tenant/stores/${targetId}/${deviceId}\`)` + `router.refresh()`. Pending state disables the confirm button and dialog close (same pending-lock pattern as `DeleteStoreDialog`).
- **Page wiring** (`app/(tenant)/tenant/stores/[storeId]/[deviceId]/page.tsx`): compute `canManage` (owner/admin membership, same pattern as the stores page) and `otherStores` = `getTenantStores(organizationId)` minus the current store, mapped to `{id, name}`. Render the control near `DevicePauseControl` only when `canManage && otherStores.length > 0`.
- No action/data/audit changes. No schema change.

## Testing

Gates (`npx tsc --noEmit`, `npm test`, `npm run build`). Live QA on prod DB (⚠️ `.env.local` = PROD; no db:seed): move a seed device (NOT Printer b580) from SoMa Roastery to Mission District via the new control — redirected URL renders, audit shows "Device reassigned" — then move it back. Member-role/no-other-store gating verified by code inspection.
