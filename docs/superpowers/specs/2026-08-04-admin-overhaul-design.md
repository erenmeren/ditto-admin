# Super-Admin Overhaul — Cleanup & Alignment

**Date:** 2026-08-04 · **Status:** Approved
**Scope decision:** cleanup + alignment with the current architecture. No new
features. Whole-page deletion allowed but not needed — the audit found no page
worth deleting. Two phases, one plan.

## Background

The tenant panel absorbed every product pivot (trigger-only 2026-07-02,
credits-only billing 2026-07-04 + dual-track plans later, eco/analytics removal
2026-07-19, MQTT-only transport 2026-07-30, archive lifecycle, factory
registry). The super-admin panel was audited on 2026-08-04; it has no dead
pages or old subsystems, but carries a thin layer of poll-era copy, invoice-era
dead code, and archive-lifecycle leaks in shared queries.

## Phase 1 — Deletions, copy fixes, behavior fixes

### 1.1 Dead code — delete outright

| Item | Location |
|---|---|
| `Placeholder` component | `components/placeholder.tsx` (zero importers) |
| `FleetTable` component | `components/fleet-table.tsx` (zero importers) |
| Seven audit **constants**: `invoicePaid`, `invoicePaymentFailed`, `invoiceVoid`, `invoiceSent`, `invoiceOverdue`, `subscriptionStatusChanged`, `billingActivated` | `lib/audit.ts:19-25` (zero emission sites) |
| `money` prop on `BreakdownBarChart` + `formatCurrency` | `components/charts.tsx:51,55,134,145`, `lib/format.ts:3` |
| `trial` branch in tenant status badge + type member | `components/tenant-status-badge.tsx:10-14`, `lib/types.ts:9` (`mapTenantStatus` never emits it) |
| `getTenantSummaries` export → internal function | `lib/data.ts:1093` (no external callers) |
| `AdminOverview.daily` computation | `lib/data.ts:1147,1155` (computed every render, never read) |
| `ExportButton` no-data toast path | `components/export-button.tsx:31-36` (only caller always passes data) |

**Exception (approved):** the seven matching **labels** in
`lib/audit-labels.ts:10-16` are KEPT with a "legacy display only" comment —
historic audit rows from before 2026-07-04 must still render friendly labels.
The display layer keeps understanding history; the code stops producing it.

### 1.2 Copy & terminology

| Location | From → To |
|---|---|
| `components/devices/command-bar.tsx:36` | "queued — the device will pick it up on its next check-in" → sent-over-MQTT wording (command is pushed immediately; failed publish already surfaces separately) |
| `app/(admin)/admin/page.tsx:43` | hardcoded `delta={12.1}` fake trend badge → **removed** (real deltas are a separate, out-of-scope feature) |
| `FileText` icon on activation KPIs | → `Zap`, three sites: `admin/page.tsx:2,45`, `customers/[tenantId]/page.tsx:2,160`, `devices/[deviceId]/page.tsx:3,87,88` |
| `lib/nav.ts:30` | "Billing & Revenue" → "Billing & Credits" (matches the page's own title) |
| `components/admin/integration-health-card.tsx:86` | "Charges and subscriptions run…" → "Charges run…" |
| `lib/data.ts:8`, `lib/data.ts:1640` | invoice-era module/section comments → describe credits/plans reality |
| `components/app-shell.tsx:130` | "printed slip" comment → neutral wording |

**Explicit non-change:** `lib/health.ts:45` alert key `"documents-stuck"` stays.
It is a persisted dedupe identity (unique index on the `alert` table); renaming
means a migration or duplicate alerts. The visible message is already correct.
Strengthen the code comment to say the key is historical and deliberate.

### 1.3 Archived-org filter fixes (behavior changes)

Four queries gain the archived-org exclusion `/admin/inventory` already applies
deliberately (`app/(admin)/admin/inventory/page.tsx:40-47`). Reuse the existing
`loadAllOrgs` default / a shared helper — do not copy-paste `isNull(archivedAt)`
into each query.

1. **`getPlatformHealth`** (`lib/data.ts:1815-1817, 1876`): fleet counts limited
   to claimed devices in non-archived orgs → `/admin/health` device count equals
   `/admin/devices` "All". `inactiveTenants` skips archived orgs.
2. **`getAlertInputs`** (`lib/data.ts:1962`): same fix; this feeds the cron
   (`lib/alerts-sync.ts:86`), so phantom "inactive tenant" alerts and their
   notification emails stop.
3. **`getCreditsOverview`** (`lib/data.ts:2216`): archived orgs leave the
   per-tenant table; frozen credits leave "Outstanding liability". Page gets a
   footnote: "Archived customers' frozen credits are excluded."
4. **`getCreditUsageAllOrgs`** (`lib/data.ts:2166-2179`): overview "Credits by
   company" excludes archived orgs.

**Existing phantom alerts:** after the fix, verify locally that `alerts-sync`'s
own resolve flow closes them. If it does not, add a one-time cleanup script
(not a migration) to the plan.

## Phase 2 — Alignment

### 2.1 Pinned QR resolution on admin device detail

`app/(admin)/admin/devices/[deviceId]/page.tsx:72` currently shows raw
`device.pinnedUrl` ("—" even when a store/tenant pin is live). Wire the existing
`getDevicePinContext` (`lib/data.ts:977`) — show the **effective** pin, its
source level (device / store / tenant / none), and `pinMode`. No new resolution
logic.

### 2.2 Tenant health badge unification

`summarize` (`lib/data.ts:322,340`) passes `stuckPendingCount` and
`lastActivityAt` to `tenantHealthLevel`, same as the detail page
(`lib/data.ts:1249`). Gather both for the page's orgs in **one grouped query
each** (no N+1), following the pattern of the existing activation counts.
Remove the "intentionally omitted" comment in `lib/tenant-health.ts:12-13`.
Result: a tenant shows the same health level in the list and the detail page.

### 2.3 billingPlan visibility

- **Customers list:** neutral plan badge column — Credits / Flat / Base+Usage
  (join on `tenantSettings.billingPlan`).
- **Customer detail:** when plan is `flat`, the Credits card gets one muted
  note — "This tenant is on the flat plan; triggers do not consume credits."
  Card and grant form stay (admin may still grant).
- **`/admin/billing`:** a plan-mix summary strip (tenant count per plan; device
  count for flat/base_usage as the subscription-revenue proxy) + a plan column
  in the per-tenant table. Credit tables unchanged.
- **Out of scope (noted as future work):** real Stripe revenue figures and
  surfacing per-tenant Stripe subscription sync status
  (`lib/actions/billing-plan.ts:52` fail-open path is invisible today).

### 2.4 Firmware page query moves to the data layer

`app/(admin)/admin/firmware/page.tsx:11-15` queries Drizzle inline — the only
admin page that does. Move to `getFirmwareReleases()` in `lib/data.ts`.
Version-adoption visibility stays out of scope.

### 2.5 Turkish super-admin manual

Targeted fixes in `docs/manuals/tr/super-admin-kilavuzu.md` (no rewrite):
polling narrative → MQTT push (`:80`, `:695`, `:723`); "printers online" →
"screens online" (`:224`); "document history" → "command history" (`:638-639`);
"Faturalandırma ve Gelir" → new nav label (`:174`); provision-dialog quotes
"printer" → "screen" (`:504-510`).

## Testing & verification

- **New vitest coverage:** the four queries exclude archived orgs; `summarize`
  and detail produce identical health levels from identical fixtures; admin pin
  resolution returns the effective pin for device-, store-, and tenant-level
  inherit scenarios.
- Existing suite (324+ tests), `tsc`, and `next build` stay green.
- Local verification of phantom-alert resolution (see 1.3).
- **No deploy:** work stays in the working tree; the user tests locally and
  batches the deploy (established workflow).

## Explicitly out of scope

Real KPI deltas on the admin overview, firmware version-adoption view, Stripe
revenue/sync surfacing, MQTT per-channel liveness on the fleet list,
serial-conflict flags on the fleet list, any new admin pages or nav entries.
