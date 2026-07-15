# Eco + Analytics Removal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the two eco cards from the tenant Reports page, delete the tenant Analytics page (UI + nav + data layer), and purge the now-dead eco subsystem.

**Architecture:** Pure deletion refactor, no new behavior and no DB changes. Ordered so the tree compiles after every task: strip UI consumers first (Reports, Analytics page), then the data layer (`getStoresAnalytics`, `CustomerDetail.eco`), then the pure helpers (`toComparisonRows`), then orphaned chart components, and finally the eco module files themselves.

**Tech Stack:** Next.js 16 App Router, TypeScript strict, vitest.

**Spec:** `docs/superpowers/specs/2026-07-15-eco-analytics-removal-design.md`

## Global Constraints

- **NO code commits.** Per the user's batch-deploy workflow, all code changes stay in the working tree; the user tests locally (dev server on port 3000) and requests a batch commit later. The verification step at the end of each task is `npx tsc --noEmit` instead of a commit.
- Do NOT touch `getStoreAnalytics` (singular — the store detail page's function), `StoreAnalytics`, `computeTrend`, `dayKeys`, `monthKeys`, `bucketsToSeries`, `countByDayKey`, `countByMonthKey`. They stay.
- All keeps/removals below were verified by grep on 2026-07-15; if you find an unexpected extra reference to a deleted symbol, stop and report rather than improvising.

---

### Task 1: Strip eco from the Reports page

**Files:**
- Modify: `app/(tenant)/tenant/reports/page.tsx`

**Interfaces:**
- Consumes: existing `getTenant`, `getTenantStoresPage`, `tenantMonthly` from `@/lib/data` (unchanged).
- Produces: a Reports page with three cards (Activations over time, By store, By device) and no eco references. Later tasks rely on this file no longer importing `@/lib/eco`, `@/components/eco-savings`, or `MetricAreaChart`.

- [ ] **Step 1: Rewrite the page without eco**

Replace the full contents of `app/(tenant)/tenant/reports/page.tsx` with:

```tsx
import { PageHeader } from "@/components/page-header";
import { ExportButton } from "@/components/export-button";
import { BreakdownBarChart, DocumentsAreaChart } from "@/components/charts";
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { getTenant, getTenantStoresPage, tenantMonthly } from "@/lib/data";
import { requireTenant } from "@/lib/session";
import { PAGE_SIZE } from "@/lib/list-params";

export default async function ReportsPage() {
  const { organizationId } = await requireTenant();
  const tenant = await getTenant(organizationId);
  const monthly = await tenantMonthly(organizationId);
  const { rows: stores, total: storeCount } = await getTenantStoresPage(organizationId, {
    q: "",
    page: 1,
    sort: "activations",
  });

  // Already sorted + capped to the first PAGE_SIZE (by activations) by the query above.
  const byStore = stores.map((s) => ({
    label: s.name.replace("Roastwell ", ""),
    value: s.activationsThisMonth,
  }));

  const byDevice = tenant.stores
    .flatMap((store) =>
      store.devices.map((d) => ({
        label: `${store.name.split(" ")[0]} · ${d.name}`,
        value: d.activationsThisMonth,
      })),
    )
    .sort((a, b) => b.value - a.value)
    .slice(0, 8);

  // Build a single CSV: a section per breakdown (monthly, by store, by device).
  const exportHeaders = ["Section", "Label", "Activations"];
  const exportRows: (string | number)[][] = [
    ...monthly.map((p) => ["Monthly", p.label, p.activations]),
    ...byStore.map((s) => ["By store", s.label, s.value]),
    ...byDevice.map((d) => ["By device", d.label, d.value]),
  ];

  return (
    <>
      <PageHeader
        title="Reports"
        description="Activations and breakdowns across your fleet."
      >
        <ExportButton
          label="Export report"
          filename={`${tenant.name.toLowerCase().replace(/\s+/g, "-")}-report.csv`}
          headers={exportHeaders}
          rows={exportRows}
        />
      </PageHeader>

      <Card>
        <CardHeader>
          <CardTitle>Activations over time</CardTitle>
          <CardDescription>Monthly activations, last 9 months</CardDescription>
        </CardHeader>
        <CardContent>
          <DocumentsAreaChart data={monthly} height={300} />
        </CardContent>
      </Card>

      <div className="grid gap-6 lg:grid-cols-2">
        <Card>
          <CardHeader>
            <CardTitle>By store</CardTitle>
            <CardDescription>Activations this month, per branch</CardDescription>
          </CardHeader>
          <CardContent>
            <BreakdownBarChart data={byStore} />
            {storeCount > PAGE_SIZE && (
              <p className="mt-2 text-xs text-muted-foreground">
                Showing top {PAGE_SIZE} stores by activations.
              </p>
            )}
          </CardContent>
        </Card>
        <Card>
          <CardHeader>
            <CardTitle>By device</CardTitle>
            <CardDescription>Top printers by activations this month</CardDescription>
          </CardHeader>
          <CardContent>
            <BreakdownBarChart data={byDevice} />
          </CardContent>
        </Card>
      </div>
    </>
  );
}
```

Changes vs the old file: dropped the `MetricAreaChart` + `EcoSavingsCard` imports, the `@/lib/eco` import, the `ecoOverTime` / `totalActivations` / `eco` computations, the bottom `lg:grid-cols-3` grid (both eco cards), and the "and eco savings" phrase in the description. Everything else is byte-identical.

- [ ] **Step 2: Verify**

Run: `npx tsc --noEmit`
Expected: clean (the pre-existing unrelated diagnostics in scratch `verify-*.ts` scripts, if any appear in the editor, are not part of `tsc --noEmit` because those files are outside the project's include set — the command itself must exit 0).

---

### Task 2: Delete the Analytics page and its nav entry

**Files:**
- Delete: `app/(tenant)/tenant/analytics/page.tsx`
- Modify: `lib/nav.ts`

**Interfaces:**
- Produces: `/tenant/analytics` route gone (404s via `app/not-found.tsx`); `TENANT_NAV` without the Analytics item. Task 3 relies on nothing importing `getStoresAnalytics` after this task.

- [ ] **Step 1: Delete the page**

```bash
rm "app/(tenant)/tenant/analytics/page.tsx"
rmdir "app/(tenant)/tenant/analytics"
```

- [ ] **Step 2: Remove the nav entry and the now-unused icon import**

In `lib/nav.ts`, delete this line from `TENANT_NAV`:

```ts
  { label: "Analytics", href: "/tenant/analytics", icon: LineChart },
```

and delete `LineChart,` from the lucide-react import block at the top (it has no other user in this file).

- [ ] **Step 3: Verify**

Run: `npx tsc --noEmit`
Expected: clean. (`getStoresAnalytics` is now unused but unused exports don't fail tsc.)

---

### Task 3: Remove `getStoresAnalytics` and `CustomerDetail.eco` from the data layer

**Files:**
- Modify: `lib/data.ts`

**Interfaces:**
- Consumes: nothing imports `getStoresAnalytics` anymore (Task 2).
- Produces: `CustomerDetail` without an `eco` field (its only consumer, `app/(admin)/admin/customers/[tenantId]/page.tsx`, never reads `.eco` — verified by grep). Task 4 relies on `lib/data.ts` no longer importing `toComparisonRows` / `StoreComparisonRow` / `computeEcoSavings`.

- [ ] **Step 1: Delete the `getStoresAnalytics` function**

Remove the whole block (currently `lib/data.ts:831-888`), from its doc comment through the closing brace:

```ts
/**
 * Cross-store comparison for the tenant Analytics page: per-store rows (activations
 * this month, trend vs last month, eco) sorted by activations, plus a
 * per-store monthly series for the comparison chart. Degrades to empty on error.
 */
export async function getStoresAnalytics(organizationId: string): Promise<{
  ...
}
```

(Everything between the end of `getStoreAnalytics` — `return { store, analytics };` / `}` — and `export async function getDevice(`.)

- [ ] **Step 2: Remove the eco field from `CustomerDetail`**

In the `CustomerDetail` interface (near `lib/data.ts:986`), delete:

```ts
  eco: ReturnType<typeof computeEcoSavings>;
```

In `getCustomerDetail`'s return object (near `lib/data.ts:1070`), delete:

```ts
    eco: computeEcoSavings(summary.activationsThisMonth),
```

- [ ] **Step 3: Clean the imports**

In the import block at the top of `lib/data.ts`:
- Delete the line `import { computeEcoSavings } from "./eco";`
- In the `from "./analytics"` import, delete `toComparisonRows,` and `type StoreComparisonRow,` (keep `bucketsToSeries`, `dayKeys`, `monthKeys`, `computeTrend`, `type BucketCount`, `type StoreAnalytics`).

Do NOT remove `count`, `gte`, or `sql` from the drizzle imports — all still used elsewhere in the file (verified: `count()`/`gte()`/`sql` appear in `loadOrg`, health, and credit queries).

- [ ] **Step 4: Fix the two stale comments**

Near line 226, change:

```ts
// Bucketing is UTC everywhere (matches the SQL date_trunc/extract used by the
// per-store analytics in getStoreAnalytics/getStoresAnalytics), so "today" and
```

to:

```ts
// Bucketing is UTC everywhere (matches the SQL date_trunc/extract used by the
// per-store analytics in getStoreAnalytics), so "today" and
```

Near line 361, change:

```ts
// the same join the per-store analytics (getStoreAnalytics/getStoresAnalytics)
// use, so org-wide and per-store series can never drift apart. Buckets outside
```

to:

```ts
// the same join the per-store analytics (getStoreAnalytics) uses, so org-wide
// and per-store series can never drift apart. Buckets outside
```

- [ ] **Step 5: Verify**

Run: `npx tsc --noEmit && grep -n "getStoresAnalytics\|computeEcoSavings\|StoreComparisonRow" lib/data.ts`
Expected: tsc clean; grep finds nothing.

---

### Task 4: Remove `toComparisonRows` from lib/analytics.ts and its tests

**Files:**
- Modify: `lib/analytics.ts`
- Test: `lib/analytics.test.ts`

**Interfaces:**
- Consumes: nothing imports `toComparisonRows` / `StoreComparisonRow` anymore (Task 3).
- Produces: `lib/analytics.ts` with no `./eco` import. Task 6 relies on this before deleting `lib/eco.ts`.

- [ ] **Step 1: Remove the tests first**

In `lib/analytics.test.ts`:
- Change line 3 from `import { computeTrend, toComparisonRows } from "./analytics";` to `import { computeTrend } from "./analytics";`
- Delete the entire `describe("toComparisonRows", ...)` block (currently lines 50–63, two `it`s).

- [ ] **Step 2: Run the test file — expect it green already**

Run: `npx vitest run lib/analytics.test.ts`
Expected: PASS (the remaining describes don't touch the deleted code). This ordering just keeps the suite green at every step; there is no red phase in a deletion.

- [ ] **Step 3: Remove the code**

In `lib/analytics.ts`:
- Delete `import { computeEcoSavings, type EcoSavings } from "./eco";`
- Delete the `StoreComparisonRow` interface (lines 62–68) and the whole `toComparisonRows` function (lines 75–87).
- In the header comment (line 3), change `everything testable (series shaping, trend %, eco) lives in this file.` to `everything testable (series shaping, trend %) lives in this file.`

- [ ] **Step 4: Verify**

Run: `npx tsc --noEmit && npx vitest run lib/analytics.test.ts`
Expected: both clean.

---

### Task 5: Remove orphaned chart components

**Files:**
- Modify: `components/charts.tsx`

**Interfaces:**
- Consumes: nothing imports `MetricAreaChart` (Task 1) or `StoreCompareChart` (Task 2) anymore.
- Produces: `charts.tsx` exporting only `DocumentsAreaChart`, `BreakdownBarChart` (+ `BreakdownDatum`).

- [ ] **Step 1: Delete the two components**

In `components/charts.tsx`:
- Delete the `MetricAreaChart` function and its doc comment (lines 109–148).
- Delete the `COMPARE_COLORS` const and the `StoreCompareChart` function with its doc comment (lines 201–255).
- In the recharts import block, delete `Line,` and `LineChart,` — they were only used by `StoreCompareChart`. Keep `Area`, `AreaChart`, `Bar`, `BarChart`, `CartesianGrid`, `Cell`, `ResponsiveContainer`, `Tooltip`, `XAxis`, `YAxis` (all still used).

- [ ] **Step 2: Verify**

Run: `npx tsc --noEmit`
Expected: clean.

---

### Task 6: Delete the eco module files and run full gates

**Files:**
- Delete: `lib/eco.ts`
- Delete: `components/eco-savings.tsx`

**Interfaces:**
- Consumes: all importers of both files removed in Tasks 1, 3, 4 (`components/eco-savings.tsx` imports `lib/eco.ts`, so they must be deleted together).

- [ ] **Step 1: Delete both files**

```bash
rm lib/eco.ts components/eco-savings.tsx
```

- [ ] **Step 2: Full gates**

Run: `npx tsc --noEmit && npx vitest run`
Expected: tsc clean; full suite passes — previous count was 289 tests / 40 files, so expect 287 tests / 40 files (2 `toComparisonRows` tests removed).

- [ ] **Step 3: Leftover sweep**

Run:

```bash
grep -rn "eco\b\|EcoSavings\|getStoresAnalytics\|StoreCompareChart\|MetricAreaChart\|toComparisonRows\|StoreComparisonRow\|PAPER_GRAMS\|/tenant/analytics" app components lib --include='*.ts' --include='*.tsx'
```

Expected: no hits. (Case-sensitive `eco\b` will not match `Record` or `second`; if a hit appears, it must be evaluated, not auto-deleted.)

- [ ] **Step 4: Do NOT commit**

Leave everything in the working tree per the batch-deploy workflow. Report the summary of changed/deleted files so the user can test on http://localhost:3000 (`/tenant/reports` shows 3 cards, sidebar has no Analytics item, `/tenant/analytics` 404s, admin customer detail unchanged).
