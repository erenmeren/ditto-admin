# Fleet-Scale Tenant Stores & Devices — Design

**Date:** 2026-07-12
**Status:** Approved (design)
**Scope:** Tenant workspace only. The admin fleet pages have the same scale
problem and will get a separate follow-up spec reusing this infrastructure.

## Problem

The tenant Stores page loads every store (and its per-store aggregates) in one
unpaginated table, and devices exist only inside store detail pages — there is
no org-wide device list. The target enterprise customer has ~2,000 stores and
~4,000 devices; the current page neither loads nor navigates at that scale.
Ops teams' top jobs (in priority order, per product owner): find a specific
store/device fast, catch problem (offline/paused) devices, and run
install/move operations (pool → store, store → store).

## Decision

Server-driven lists with URL-state filters (approach A of the brainstorm):
search, status filter, and page number live in the URL; server components
fetch exactly one page (50 rows) per request; totals and aggregates are
computed in SQL. A new top-level tenant **Devices** page owns the org-wide
fleet view; **Stores** slims down to a searchable, paginated branch directory.

Rejected: client-side data grid (ships the whole dataset — doesn't solve the
scale problem); command-palette-only (doesn't solve browsing/pagination; may
complement later).

## Information architecture

- Sidebar (tenant workspace): add **Devices** directly under Stores.
  Nav is defined inside `AppShell` (client) — extend the tenant nav list there.
- **Stores** (`/tenant/stores?q=&page=`): search by store name/address +
  pagination. Columns unchanged (Store, Address, Printers x/y, Activations
  (mo.), Status, row actions). Store row → existing store detail page.
  The "Unassigned devices" section moves OFF this page (see Devices).
  Header description keeps the org-wide totals (stores · online/total
  printers) — computed by one aggregate query, not by loading all rows.
- **Devices** (`/tenant/devices?q=&status=&page=`): org-wide claimed devices.
  - Status tabs: **All / Online / Offline / Paused / Pool**. Online/Offline/
    Paused reflect physical state and INCLUDE pool devices; Pool filters by
    location (`storeId IS NULL AND claimedAt IS NOT NULL`). Tab counts follow
    the same semantics.
  - One search box matching device name OR serial OR store name (ILIKE).
  - Columns: Device (name + serial), Store ("—" for pool), Status (badge +
    last seen), Actions.
  - Row behavior: assigned device links to the existing device detail route
    (`/tenant/stores/[storeId]/[deviceId]`); pool devices have no detail page
    (existing, intended behavior) and get an inline **Assign to store**
    action. Assigned devices get an inline **Move to store** action
    (tenant analog of the admin row action; both reuse
    `assignDeviceToStore`).
- No store-filter dropdown in v1 — a 2,000-entry combobox is its own project,
  and search already matches store names (YAGNI).

## Data layer (`lib/data.ts` — the single data seam)

Two new paginated functions; each returns one page plus the total in a single
SQL round trip (`count(*) over ()`); page size 50 (module constant).

- `getTenantStoresPage(organizationId, { q, page })` →
  `{ rows: StoreListRow[], total: number }` where `StoreListRow` matches
  today's store table columns (id, name, address, deviceCount, onlineCount,
  activationsThisMonth, status). One grouped query: `store` LEFT JOIN device
  aggregates LEFT JOIN this-month acked-trigger counts. The activation
  definition is byte-identical to today's (`device_command` where
  `type='trigger' AND status='acked'`, current month, same month-boundary
  convention as the existing implementation — read it and copy it).
- `getTenantDevicesPage(organizationId, { q, status, page })` →
  `{ rows: DeviceListRow[], total: number, counts: TabCounts }` where rows
  carry id, name, serial, storeId, storeName (null for pool), effective
  status, lastSeen ISO, and `counts` holds the five tab totals for the
  current `q`. Claimed devices only (`claimedAt IS NOT NULL`).
- Status semantics: filter and display use the STORED `device.status` column —
  exactly what every existing tenant page shows (`buildTenant` maps the raw
  column; the daily reconcile cron flips stale rows offline). Re-deriving an
  effective status here would make the fleet list disagree with the store
  detail pages.
- Search: `ILIKE '%' || q || '%'` with `%`/`_`/`\` escaped in `q` first.
  No new indexes or extensions: per-org row counts (~2K/4K) under the
  existing `organizationId` indexes make ILIKE scans cheap.
- Out-of-range page → empty rows + honest total (no clamping/redirect).

## URL/state contract (pure helper `lib/list-params.ts`)

- `parseListParams(searchParams)` → `{ q: string, status: DeviceStatusFilter,
  page: number }`; `page` ≥ 1 integer (default 1), `status` one of
  `all|online|offline|paused|pool` (default `all`, invalid → `all`), `q`
  trimmed (length cap 100).
- `pageCount(total, pageSize)` and `escapeLike(q)` live here too.
- Fully unit-tested (TDD) — this is the testable core of the feature.

## UI components

- `components/list-controls.tsx` (client, reused by both pages): debounced
  (300 ms) search input + optional status tabs. Writes `q`/`status` to the
  URL via `router.replace`, always resetting `page`. Reads initial values
  from props (server passes the parsed params).
- `components/pagination-bar.tsx` (server-friendly): `‹ Previous · Page X of
  Y (N total) · Next ›` as plain `<Link>`s preserving current query params;
  hidden when only one page.
- Pages stay server components: read `searchParams`, call the data function,
  render. Layout follows the CLAUDE.md dashboard rhythm (PageHeader, fragment
  return, `space-y` conventions).
- Devices page actions (assign for pool rows, move for assigned rows) share
  one `StorePickerDialog`: a searchable Command-combobox of store names —
  a plain 2,000-entry Select would recreate the scale problem inside the
  dialog. Options (id + name only) load lazily via a tenant-gated server
  action when the dialog opens; selection calls the existing
  `assignDeviceToStore` (requireTenant + owner/admin). The per-store-page
  controls (`UnassignedDevices` on Stores today, `DeviceMoveControl` on the
  device detail page) keep their current UX; `UnassignedDevices` is deleted
  together with its Stores-page section when the Devices page ships.
- Empty states: "No stores match", "No devices match", per-tab phrasing for
  an empty pool ("No unassigned devices").

## Permissions

Unchanged: viewing is any member; assign/move actions render only for
owner/admin (same `canManage` gate the pages already compute).

## Edge cases

- Debounce keeps typing smooth; Enter submits immediately.
- Tab counts and table rows always agree (both derive from the same filtered
  query parameters).
- Behavior for today's small orgs is visually unchanged: one page, no
  pagination bar, same columns.
- No migration; code-only. `UnassignedDevices` disappears from Stores in the
  same release that Devices ships (no orphaned duplicate UI).

## Testing

- TDD for `lib/list-params.ts` (parsing, clamping, escaping, page math).
- Data functions follow repo convention (DB modules not unit-tested); their
  correctness is exercised by live QA: search by name/serial/store, each
  status tab, pool assign, move, pagination links, empty states — on prod
  with the Starbucks org (create a few extra stores for pagination if needed,
  clean them up after).
- Full gates per task: `npx tsc --noEmit && npm test && npm run build`.

## Out of scope

- Admin fleet pages (follow-up spec; will reuse list-params + the two shared
  components).
- ⌘K global search / command palette.
- Bulk multi-select operations (assign N devices at once).
- Store-filter dropdown/combobox.
- CSV export.
