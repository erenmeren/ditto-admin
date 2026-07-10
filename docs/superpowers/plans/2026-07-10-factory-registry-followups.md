# Factory Registry Follow-ups Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Close the final-review follow-up list from the factory-registry ship (2026-07-10): factory-scale import + pagination, small hardening items, and the ops trio (rate-limit purge, auto-claim admin email, hijack-recovery runbook).

**Architecture:** No new subsystems — each task extends an existing seam (data layer, server actions, inventory UI, cron, docs) following the repo's established precedent for that seam (audit-log pagination, health-cron shape, alerts-sync email pattern).

**Tech Stack:** existing stack; zod v4 is already a dependency.

**Parent spec:** `docs/superpowers/specs/2026-07-09-factory-registry-provisioning-design.md` (invariants unchanged — serial never authenticates; auto-claim one-shot allocated+org+store only).

## Global Constraints

- TypeScript strict; Next.js 16 (check `node_modules/next/dist/docs/` for unfamiliar APIs); shadcn radix-nova only.
- Serial normal form: 12 lowercase hex, no separators; validate via existing `normalizeSerial`.
- All inventory actions stay platform-admin-gated (`requirePlatformAdmin()` first) and return `{ ok: boolean, ... }` shapes — invalid input returns `ok:false`, never throws to the client.
- Keep the coalescing re-import semantics added in bb0ba13 (null incoming metadata never wipes existing values).
- `npm run build && npx tsc --noEmit && npm test` clean before every commit; new pure logic gets vitest coverage.
- Commit per task on main.

---

### Task F1: Factory-scale import — chunked upsert + single-add

**Files:** Modify `lib/factory-registry.ts` (importFactoryDevices), `lib/actions/inventory.ts` (+`addSerialAction`), `components/inventory/inventory-table.tsx` (single-add input in the import card). Test: `lib/factory-registry-csv.test.ts` untouched; add `lib/chunk.test.ts` if a pure chunk helper is created.

**Requirements:**
1. `importFactoryDevices` must batch: one multi-row `INSERT ... ON CONFLICT DO UPDATE` per chunk of 500 rows instead of one round trip per row (10k rows = 20 statements, not 10k). Preserve the coalesce(excluded.x, existing.x) semantics exactly. A pure `chunk<T>(arr, size)` helper is fine — unit-test it.
2. New server action `addSerialAction(serial: string, batchCode?: string | null, hardwareRevision?: string | null): Promise<{ ok: boolean; error?: string }>` — platform-admin, normalize+validate serial (reject with `ok:false` + message on invalid), upsert one row (same coalescing import path — reuse `importFactoryDevices` with a single row).
3. UI: in the "Import from factory CSV" card, add a compact inline form (Input for serial + optional batch Input + "Add" Button) — barcode scanners type the serial and send Enter, so submitting on Enter must work (wrap in a `<form onSubmit>`). Success/error toasts; refresh happens via the action's `revalidatePath`.

**Produces (F2/F3 depend on):** unchanged `importFactoryDevices` signature; new `addSerialAction` export.

### Task F2: Inventory pagination + store visibility

**Files:** Modify `lib/factory-registry.ts` (`getFactoryDevices` → paged), `app/(admin)/admin/inventory/page.tsx`, `components/inventory/inventory-table.tsx`.

**Requirements:**
1. Follow the `getOrgAuditPage` precedent (lib/data.ts:1024): `getFactoryDevicePage(opts: { page: number; pageSize?: number (default 50); status?: RegistryStatus | "all"; batch?: string })` returning `{ rows: InventoryRow[]; total: number; page; pageSize; pageCount }` with the same over-range clamping. Status/batch filtering moves SERVER-side (`ilike` for batch substring). Keep a cheap grouped-count query for the KPI tiles (`select status, count(*) group by status`) so KPIs reflect the whole registry, not the page.
2. Page reads `searchParams` (`page`, `status`, `batch`) — filters become URL state; the client table's Select/Input now navigate via `router.replace` with the updated query (debounce the batch input ~300ms). Pagination controls: Prev/Next + "Page X of Y" (match the tenant Activity table's pattern — check `app/(tenant)/tenant/activity/` before writing).
3. New "Store" column between Customer and Device: show allocated store name (needs a leftJoin on store in the page query) or "—"; when `status === "allocated"` and a store is set, render a small outline badge "zero-touch" next to the store name (this is the security-relevant armed state).
4. `getFactoryDevices()` (unpaged) may be deleted if nothing else imports it — check first.

### Task F3: Hardening trio (UI catches, zod guards, deallocation audit)

**Files:** Modify `components/inventory/inventory-table.tsx`, `lib/actions/inventory.ts`, `lib/factory-registry.ts` (deallocate needs org snapshot).

**Requirements:**
1. `onImportFile` and `onAllocate` get `catch` blocks with `toast.error("Import failed — try again.")` / `toast.error("Allocation failed — try again.")` (matching the deallocate/RMA handlers added in 6e91bb6).
2. zod v4 input schemas at the top of `lib/actions/inventory.ts` for every action's args (serial → refine via `normalizeSerial` returning the normalized value; serials arrays non-empty, each valid; status literal union "rma"|"retired"; csvText string). Parse with `safeParse`; on failure return `{ ok: false, error: "Invalid input." }`-shaped results consistent with each action's return type. No behavioral change for valid input.
3. Deallocation audit: `deallocateSerials` in the data layer returns the affected rows' previous `allocatedOrganizationId` (change return to `{ updated: number; byOrg: Record<string, string[]> }` — orgId → serials). The action then records `AUDIT.registryDeallocated` once per org (actor = admin user, metadata `{ count, serials }`). This activates the previously-dead constant; keep the code comment updated.

### Task F4: Ops — rate-limit purge, auto-claim admin email, hijack runbook

**Files:** Modify `app/api/cron/health/route.ts` (or better: `lib/alerts-sync.ts` if the purge fits its summary), `app/api/device/claim/route.ts`, `lib/factory-registry.ts` (only if the email needs data the route lacks). Create `docs/runbooks/factory-registry-hijack-recovery.md`.

**Requirements:**
1. Purge: in the daily health cron, after alert evaluation, `DELETE FROM rate_limit WHERE window_start < now() - interval '24 hours'` (drizzle `sql` — mind the Date-serialization trap fixed in 4fef7a5: use a SQL-side interval, no JS Date param). Include `purgedRateLimitRows` in the cron's JSON summary.
2. Auto-claim email: on the auto-claim success path in the claim route, fire a best-effort platform-admin notification inside `after(...)` (Next's after — see the webhook pattern noted in lib docs / alerts-sync admin email at lib/alerts-sync.ts:137 for how platform admins are enumerated). Subject like `Device auto-claimed: <serial>`; body: serial, org name, device id, timestamp, one line linking the runbook. Must never affect the response (sendEmail is already best-effort; RESEND unset in prod → silently no-ops, that is fine and expected).
3. Runbook `docs/runbooks/factory-registry-hijack-recovery.md`: explain the allocated-window exposure (public serial + any well-formed code mints the org's key until install), how to detect (audit `device.auto_claimed` events, unexpected auto-claims, `serialConflict`/`unregistered` badges), and the 3-step recovery: (1) delete the rogue device row (admin device detail → delete), (2) revert the registry row to `allocated` (today: direct DB update — state this honestly and show the SQL; a UI revert action is future work), (3) re-run the legitimate install. Note the deterrents (one-shot transition, rate limits, audit trail).

---

## Verification wave

After F1-F4: full `npm run build && npm test && npm run lint`, dev-DB live checks (chunked import of ~1200 rows then cleanup; single-add; paginated page 2; deallocate audit row visible), then wave review → push → `vercel --prod --yes` → prod smoke.
