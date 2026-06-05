# Per-Store Analytics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give tenants per-store analytics — a single-store deep view (trends, revenue, eco, busiest day/peak hour) on the store detail page, and a new `/tenant/analytics` page comparing all stores.

**Architecture:** Hybrid aggregation — thin Drizzle `GROUP BY` queries in `lib/data.ts` return compact per-store/per-bucket counts; all derivation logic (series shaping, trend %, peak selection, revenue, eco) lives in a new pure, unit-tested `lib/analytics.ts`. UI reuses existing chart/KPI/export components plus one new multi-line chart.

**Tech Stack:** Next 16 App Router, TypeScript, Drizzle (`sql`/`count`/`groupBy` aggregates on the `receipt` table), recharts, vitest (`lib/**/*.test.ts`).

---

## File Structure

- **Create `lib/analytics.ts`** — pure helpers + types (keys, series, trend, peaks, comparison).
- **Create `lib/analytics.test.ts`** — unit tests for all pure helpers.
- **Modify `lib/data.ts`** — add `getStoreAnalytics` + `getStoresAnalytics` (IO; call the pure helpers).
- **Modify `components/charts.tsx`** — add `StoreCompareChart` (multi-line per store).
- **Modify `app/(tenant)/tenant/stores/[storeId]/page.tsx`** — add the analytics section.
- **Create `app/(tenant)/tenant/analytics/page.tsx`** — the comparison page.
- **Modify `lib/nav.ts`** — add the "Analytics" nav item.

---

## Task 1: Pure helpers — keys + series (TDD)

**Files:**
- Create: `lib/analytics.ts`
- Test: `lib/analytics.test.ts`

- [ ] **Step 1: Write the failing test**

Create `lib/analytics.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { dayKeys, monthKeys, bucketsToSeries } from "./analytics";

const NOW = new Date("2026-06-05T12:00:00Z");

describe("dayKeys", () => {
  it("returns n UTC day keys ending today, oldest first", () => {
    expect(dayKeys(NOW, 3)).toEqual([
      { key: "2026-06-03", label: "Jun 3" },
      { key: "2026-06-04", label: "Jun 4" },
      { key: "2026-06-05", label: "Jun 5" },
    ]);
  });
});

describe("monthKeys", () => {
  it("returns n UTC month keys ending this month, oldest first", () => {
    expect(monthKeys(NOW, 3)).toEqual([
      { key: "2026-04", label: "Apr" },
      { key: "2026-05", label: "May" },
      { key: "2026-06", label: "Jun" },
    ]);
  });
});

describe("bucketsToSeries", () => {
  it("zero-fills missing buckets and computes revenue", () => {
    const keys = dayKeys(NOW, 3);
    const series = bucketsToSeries([{ bucket: "2026-06-04", count: 5 }], keys, 4);
    expect(series).toEqual([
      { label: "Jun 3", receipts: 0, revenue: 0 },
      { label: "Jun 4", receipts: 5, revenue: 20 },
      { label: "Jun 5", receipts: 0, revenue: 0 },
    ]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run lib/analytics.test.ts`
Expected: FAIL — `Cannot find module './analytics'`.

- [ ] **Step 3: Write the implementation**

Create `lib/analytics.ts`:

```ts
// Pure (IO-free) analytics derivations for per-store reporting. The data layer
// (lib/data.ts) runs thin SQL GROUP BY queries and feeds the compact count rows
// here; everything testable (series shaping, trend %, peak selection, revenue,
// eco) lives in this file. Bucketing is UTC, consistent across keys + SQL.

import type { TimePoint } from "./types";
import { computeEcoSavings, type EcoSavings } from "./eco";

/** A grouped count row from SQL: a bucket key (day "YYYY-MM-DD" or month "YYYY-MM"). */
export interface BucketCount { bucket: string; count: number }
export interface DowCount { dow: number; count: number }   // dow 0..6 (Sun..Sat, Postgres)
export interface HourCount { hour: number; count: number } // hour 0..23
export interface BucketKey { key: string; label: string }

const round2 = (n: number) => Math.round(n * 100) / 100;

/** n UTC day keys ending on `now`'s date, oldest first. */
export function dayKeys(now: Date, n: number): BucketKey[] {
  const base = Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate());
  const keys: BucketKey[] = [];
  for (let i = n - 1; i >= 0; i--) {
    const d = new Date(base - i * 86_400_000);
    keys.push({
      key: d.toISOString().slice(0, 10),
      label: d.toLocaleDateString("en-US", { month: "short", day: "numeric", timeZone: "UTC" }),
    });
  }
  return keys;
}

/** n UTC month keys ending on `now`'s month, oldest first. */
export function monthKeys(now: Date, n: number): BucketKey[] {
  const y = now.getUTCFullYear();
  const m = now.getUTCMonth();
  const keys: BucketKey[] = [];
  for (let i = n - 1; i >= 0; i--) {
    const d = new Date(Date.UTC(y, m - i, 1));
    keys.push({
      key: `${d.getUTCFullYear()}-${String(d.getUTCMonth() + 1).padStart(2, "0")}`,
      label: d.toLocaleDateString("en-US", { month: "short", timeZone: "UTC" }),
    });
  }
  return keys;
}

/** Join grouped counts onto the expected ordered keys, zero-filling gaps. */
export function bucketsToSeries(counts: BucketCount[], keys: BucketKey[], price: number): TimePoint[] {
  const byKey = new Map(counts.map((c) => [c.bucket, c.count]));
  return keys.map((k) => {
    const receipts = byKey.get(k.key) ?? 0;
    return { label: k.label, receipts, revenue: round2(receipts * price) };
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run lib/analytics.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add lib/analytics.ts lib/analytics.test.ts
git commit -m "feat(analytics): pure day/month keys + zero-filled series helper"
```

---

## Task 2: Pure helpers — trend, peaks, comparison (TDD)

**Files:**
- Modify: `lib/analytics.ts`
- Modify: `lib/analytics.test.ts`

- [ ] **Step 1: Add the failing tests**

Append to `lib/analytics.test.ts` (add the new names to the existing import from `./analytics`):

```ts
import {
  computeTrend,
  dowLabel,
  hourLabel,
  pickPeakDow,
  pickPeakHour,
  buildPeak,
  toComparisonRows,
} from "./analytics";

describe("computeTrend", () => {
  it("computes percent change", () => {
    expect(computeTrend(10, 5)).toEqual({ current: 10, previous: 5, pctChange: 100 });
    expect(computeTrend(5, 10)).toEqual({ current: 5, previous: 10, pctChange: -50 });
  });
  it("returns null pctChange when previous is 0", () => {
    expect(computeTrend(5, 0).pctChange).toBeNull();
    expect(computeTrend(0, 0).pctChange).toBeNull();
  });
});

describe("dowLabel / hourLabel", () => {
  it("labels days of week", () => {
    expect(dowLabel(0)).toBe("Sundays");
    expect(dowLabel(6)).toBe("Saturdays");
  });
  it("labels hour ranges across am/pm boundaries", () => {
    expect(hourLabel(0)).toBe("12–1am");
    expect(hourLabel(12)).toBe("12–1pm");
    expect(hourLabel(23)).toBe("11pm–12am");
  });
});

describe("pickPeakDow / pickPeakHour", () => {
  it("returns the argmax (first on ties)", () => {
    expect(pickPeakDow([{ dow: 1, count: 3 }, { dow: 6, count: 9 }])).toEqual({ dow: 6, count: 9, label: "Saturdays" });
    expect(pickPeakHour([{ hour: 9, count: 2 }, { hour: 12, count: 8 }])).toEqual({ hour: 12, count: 8, label: "12–1pm" });
  });
  it("returns nulls when empty or all-zero", () => {
    expect(pickPeakDow([])).toEqual({ dow: null, count: 0, label: null });
    expect(pickPeakHour([{ hour: 3, count: 0 }])).toEqual({ hour: null, count: 0, label: null });
  });
});

describe("buildPeak", () => {
  it("composes dow + hour peaks", () => {
    expect(buildPeak([{ dow: 6, count: 9 }], [{ hour: 12, count: 8 }])).toEqual({
      busiestDow: 6, busiestDowLabel: "Saturdays", busiestDowCount: 9,
      peakHour: 12, peakHourLabel: "12–1pm", peakHourCount: 8,
    });
  });
});

describe("toComparisonRows", () => {
  it("maps + sorts by receipts desc", () => {
    const rows = toComparisonRows([
      { storeId: "a", storeName: "A", current: 3, previous: 2, price: 4 },
      { storeId: "b", storeName: "B", current: 10, previous: 5, price: 4 },
    ]);
    expect(rows.map((r) => r.storeId)).toEqual(["b", "a"]);
    expect(rows[0].trend.pctChange).toBe(100);
    expect(rows[0].revenueThisMonth).toBe(40);
    expect(rows[0].eco.receipts).toBe(10);
  });
  it("returns [] for empty input", () => {
    expect(toComparisonRows([])).toEqual([]);
  });
});
```

- [ ] **Step 2: Run tests to verify the new ones fail**

Run: `npx vitest run lib/analytics.test.ts`
Expected: FAIL — `computeTrend` (and the others) not exported.

- [ ] **Step 3: Add the implementation**

Append to `lib/analytics.ts`:

```ts
export interface Trend {
  current: number;
  previous: number;
  /** Percent change vs previous; null when previous is 0 (undefined ratio). */
  pctChange: number | null;
}
export interface Peak {
  busiestDow: number | null;
  busiestDowLabel: string | null;
  busiestDowCount: number;
  peakHour: number | null;
  peakHourLabel: string | null;
  peakHourCount: number;
}
export interface StoreAnalytics {
  daily: TimePoint[];
  monthly: TimePoint[];
  monthTrend: Trend;
  revenueThisMonth: number;
  eco: EcoSavings;
  peak: Peak;
}
export interface StoreComparisonRow {
  storeId: string;
  storeName: string;
  receiptsThisMonth: number;
  trend: Trend;
  revenueThisMonth: number;
  eco: EcoSavings;
}

const DOW_LABELS = ["Sundays", "Mondays", "Tuesdays", "Wednesdays", "Thursdays", "Fridays", "Saturdays"];

export function computeTrend(current: number, previous: number): Trend {
  const pctChange = previous === 0 ? null : Math.round(((current - previous) / previous) * 100);
  return { current, previous, pctChange };
}

export function dowLabel(dow: number): string {
  return DOW_LABELS[dow] ?? "";
}

export function hourLabel(hour: number): string {
  const to12 = (h: number) => ({
    base: h % 12 === 0 ? 12 : h % 12,
    period: h < 12 ? "am" : "pm",
  });
  const start = to12(hour);
  const end = to12((hour + 1) % 24);
  return start.period === end.period
    ? `${start.base}–${end.base}${end.period}`
    : `${start.base}${start.period}–${end.base}${end.period}`;
}

export function pickPeakDow(rows: DowCount[]): { dow: number | null; count: number; label: string | null } {
  let best: DowCount | null = null;
  for (const r of rows) if (!best || r.count > best.count) best = r;
  if (!best || best.count === 0) return { dow: null, count: 0, label: null };
  return { dow: best.dow, count: best.count, label: dowLabel(best.dow) };
}

export function pickPeakHour(rows: HourCount[]): { hour: number | null; count: number; label: string | null } {
  let best: HourCount | null = null;
  for (const r of rows) if (!best || r.count > best.count) best = r;
  if (!best || best.count === 0) return { hour: null, count: 0, label: null };
  return { hour: best.hour, count: best.count, label: hourLabel(best.hour) };
}

export function buildPeak(dowRows: DowCount[], hourRows: HourCount[]): Peak {
  const d = pickPeakDow(dowRows);
  const h = pickPeakHour(hourRows);
  return {
    busiestDow: d.dow, busiestDowLabel: d.label, busiestDowCount: d.count,
    peakHour: h.hour, peakHourLabel: h.label, peakHourCount: h.count,
  };
}

export function toComparisonRows(
  input: Array<{ storeId: string; storeName: string; current: number; previous: number; price: number }>,
): StoreComparisonRow[] {
  return input
    .map((s) => ({
      storeId: s.storeId,
      storeName: s.storeName,
      receiptsThisMonth: s.current,
      trend: computeTrend(s.current, s.previous),
      revenueThisMonth: round2(s.current * s.price),
      eco: computeEcoSavings(s.current),
    }))
    .sort((a, b) => b.receiptsThisMonth - a.receiptsThisMonth);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run lib/analytics.test.ts`
Expected: PASS (all describes green).

- [ ] **Step 5: Commit**

```bash
git add lib/analytics.ts lib/analytics.test.ts
git commit -m "feat(analytics): trend, peak day/hour, and store-comparison helpers"
```

---

## Task 3: Data layer — `getStoreAnalytics` (IO)

**Files:**
- Modify: `lib/data.ts`

- [ ] **Step 1: Add imports**

In `lib/data.ts`, change the drizzle import (line ~13) to add `sql`:

```ts
import { and, count, desc, eq, gte, isNotNull, lt, lte, max, ne, sql } from "drizzle-orm";
```

Add an import for the analytics helpers (after the `computeEcoSavings` import at line ~29):

```ts
import {
  bucketsToSeries,
  dayKeys,
  monthKeys,
  computeTrend,
  buildPeak,
  toComparisonRows,
  type StoreAnalytics,
  type StoreComparisonRow,
} from "./analytics";
```

- [ ] **Step 2: Add `getStoreAnalytics`**

Add this function to `lib/data.ts` immediately after the existing `getStore` function (around line 365):

```ts
/**
 * Per-store analytics: daily/monthly receipt series, this-vs-last-month trend,
 * revenue + eco for this month, and busiest day-of-week / peak hour. Returns the
 * store too so the page can render without a second lookup. null if not found.
 */
export async function getStoreAnalytics(
  storeId: string,
): Promise<{ store: Store; analytics: StoreAnalytics } | null> {
  const result = await getStore(storeId);
  if (!result) return null;
  const { store, tenant } = result;
  const price = tenant.perPrintPrice;
  const now = new Date();

  const since30 = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate() - 29));
  const since9mo = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth() - 8, 1));
  const since90 = new Date(now.getTime() - 90 * 86_400_000);

  const dayExpr = sql<string>`to_char(date_trunc('day', ${receiptTable.createdAt}), 'YYYY-MM-DD')`;
  const monthExpr = sql<string>`to_char(date_trunc('month', ${receiptTable.createdAt}), 'YYYY-MM')`;
  const dowExpr = sql<number>`extract(dow from ${receiptTable.createdAt})::int`;
  const hourExpr = sql<number>`extract(hour from ${receiptTable.createdAt})::int`;
  const scoped = (since: Date) =>
    and(eq(receiptTable.storeId, storeId), gte(receiptTable.createdAt, since));

  const [dailyRows, monthlyRows, dowRows, hourRows] = await Promise.all([
    db.select({ bucket: dayExpr, count: count() }).from(receiptTable).where(scoped(since30)).groupBy(dayExpr),
    db.select({ bucket: monthExpr, count: count() }).from(receiptTable).where(scoped(since9mo)).groupBy(monthExpr),
    db.select({ dow: dowExpr, count: count() }).from(receiptTable).where(scoped(since90)).groupBy(dowExpr),
    db.select({ hour: hourExpr, count: count() }).from(receiptTable).where(scoped(since90)).groupBy(hourExpr),
  ]);

  const daily = bucketsToSeries(dailyRows, dayKeys(now, 30), price);
  const monthly = bucketsToSeries(monthlyRows, monthKeys(now, 9), price);
  const thisMonth = monthly[monthly.length - 1]?.receipts ?? 0;
  const lastMonth = monthly[monthly.length - 2]?.receipts ?? 0;

  const analytics: StoreAnalytics = {
    daily,
    monthly,
    monthTrend: computeTrend(thisMonth, lastMonth),
    revenueThisMonth: Math.round(thisMonth * price * 100) / 100,
    eco: computeEcoSavings(thisMonth),
    peak: buildPeak(dowRows, hourRows),
  };
  return { store, analytics };
}
```

- [ ] **Step 3: Typecheck**

Run: `npx tsc --noEmit`
Expected: no errors. (`getStore`'s return type provides `store: Store` + `tenant: Tenant`; `tenant.perPrintPrice` is a number.)

- [ ] **Step 4: Commit**

```bash
git add lib/data.ts
git commit -m "feat(data): getStoreAnalytics — per-store series, trend, peaks"
```

---

## Task 4: Data layer — `getStoresAnalytics` (IO)

**Files:**
- Modify: `lib/data.ts`

- [ ] **Step 1: Add `getStoresAnalytics`**

Add this function immediately after `getStoreAnalytics` in `lib/data.ts`:

```ts
/**
 * Cross-store comparison for the tenant Analytics page: per-store rows (receipts
 * this month, trend vs last month, revenue, eco) sorted by receipts, plus a
 * per-store monthly series for the comparison chart. Degrades to empty on error.
 */
export async function getStoresAnalytics(organizationId: string): Promise<{
  rows: StoreComparisonRow[];
  monthlyByStore: { storeId: string; storeName: string; monthly: TimePoint[] }[];
}> {
  try {
    const tenant = await getTenant(organizationId);
    const price = tenant.perPrintPrice;
    const stores = tenant.stores;
    if (stores.length === 0) return { rows: [], monthlyByStore: [] };

    const now = new Date();
    const since9mo = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth() - 8, 1));
    const monthExpr = sql<string>`to_char(date_trunc('month', ${receiptTable.createdAt}), 'YYYY-MM')`;

    const perStoreMonth = await db
      .select({ storeId: receiptTable.storeId, bucket: monthExpr, count: count() })
      .from(receiptTable)
      .where(and(eq(receiptTable.organizationId, organizationId), gte(receiptTable.createdAt, since9mo)))
      .groupBy(receiptTable.storeId, monthExpr);

    const keys = monthKeys(now, 9);
    const thisKey = keys[keys.length - 1].key;
    const lastKey = keys[keys.length - 2].key;

    const rows = toComparisonRows(
      stores.map((s) => ({
        storeId: s.id,
        storeName: s.name,
        current: perStoreMonth.find((r) => r.storeId === s.id && r.bucket === thisKey)?.count ?? 0,
        previous: perStoreMonth.find((r) => r.storeId === s.id && r.bucket === lastKey)?.count ?? 0,
        price,
      })),
    );

    const monthlyByStore = stores.map((s) => ({
      storeId: s.id,
      storeName: s.name,
      monthly: bucketsToSeries(
        perStoreMonth.filter((r) => r.storeId === s.id).map((r) => ({ bucket: r.bucket, count: r.count })),
        keys,
        price,
      ),
    }));

    return { rows, monthlyByStore };
  } catch (err) {
    console.error("[data] getStoresAnalytics failed", err);
    return { rows: [], monthlyByStore: [] };
  }
}
```

- [ ] **Step 2: Typecheck**

Run: `npx tsc --noEmit`
Expected: no errors. (`receiptTable.storeId` is `string | null`; `.find` by `s.id` never matches null rows, so null-store receipts are excluded — intended.)

- [ ] **Step 3: Commit**

```bash
git add lib/data.ts
git commit -m "feat(data): getStoresAnalytics — cross-store comparison + monthly series"
```

---

## Task 5: `StoreCompareChart` component

**Files:**
- Modify: `components/charts.tsx`

- [ ] **Step 1: Add the component**

Append to `components/charts.tsx` (it already imports `LineChart`, `Line`, `CartesianGrid`, `XAxis`, `YAxis`, `Tooltip`, `ResponsiveContainer`, and `TimePoint`):

```tsx
const COMPARE_COLORS = [
  "var(--chart-1)",
  "var(--chart-2)",
  "var(--chart-3)",
  "var(--chart-4)",
  "var(--chart-5)",
];

/** Multi-line monthly receipts comparison — one line per store. */
export function StoreCompareChart({
  data,
  height = 300,
}: {
  data: { storeId: string; storeName: string; monthly: TimePoint[] }[];
  height?: number;
}) {
  if (data.length === 0 || (data[0]?.monthly.length ?? 0) === 0) {
    return (
      <div
        className="flex items-center justify-center text-sm text-muted-foreground"
        style={{ height }}
      >
        No data yet.
      </div>
    );
  }
  // Merge per-store series into rows keyed by month label: { label, [storeId]: receipts }.
  const rows = data[0].monthly.map((point, i) => {
    const row: Record<string, string | number> = { label: point.label };
    for (const s of data) row[s.storeId] = s.monthly[i]?.receipts ?? 0;
    return row;
  });
  return (
    <ResponsiveContainer width="100%" height={height}>
      <LineChart data={rows} margin={{ top: 8, right: 8, left: -8, bottom: 0 }}>
        <CartesianGrid vertical={false} stroke="var(--border)" strokeDasharray="3 3" />
        <XAxis dataKey="label" {...AXIS} minTickGap={16} />
        <YAxis {...AXIS} width={40} />
        <Tooltip content={<ChartTooltip unit="receipts" />} cursor={{ stroke: "var(--border)" }} />
        {data.map((s, i) => (
          <Line
            key={s.storeId}
            type="monotone"
            dataKey={s.storeId}
            name={s.storeName}
            stroke={COMPARE_COLORS[i % COMPARE_COLORS.length]}
            strokeWidth={2}
            dot={false}
            activeDot={{ r: 4, strokeWidth: 0 }}
          />
        ))}
      </LineChart>
    </ResponsiveContainer>
  );
}
```

- [ ] **Step 2: Typecheck**

Run: `npx tsc --noEmit`
Expected: no errors. (`AXIS` and `ChartTooltip` are module-level in `charts.tsx`.)

- [ ] **Step 3: Commit**

```bash
git add components/charts.tsx
git commit -m "feat(charts): StoreCompareChart — multi-line per-store monthly comparison"
```

---

## Task 6: Store detail page — analytics section

**Files:**
- Modify: `app/(tenant)/tenant/stores/[storeId]/page.tsx`

- [ ] **Step 1: Swap `getStore` for `getStoreAnalytics` + add imports**

In `app/(tenant)/tenant/stores/[storeId]/page.tsx`:

Change the data import line:

```ts
import { getStoreAnalytics, getUnclaimedDevices } from "@/lib/data";
```

Add to the lucide import (the existing line imports several icons — add `CalendarClock` and `Clock`):

```ts
import { ArrowLeft, CalendarClock, Clock, Cpu, MapPin, Receipt, ReceiptText, Router, TrendingUp } from "lucide-react";
```

Add chart + format imports near the other component imports:

```ts
import { ReceiptsAreaChart } from "@/components/charts";
import { formatCurrency } from "@/lib/format";
```

- [ ] **Step 2: Replace the `getStore` call**

Replace:

```ts
  const result = await getStore(storeId);
  if (!result || result.tenant.id !== organizationId) notFound();

  const { store } = result;
```

with:

```ts
  const result = await getStoreAnalytics(storeId);
  if (!result || result.store.organizationId !== organizationId) notFound();

  const { store, analytics } = result;
```

- [ ] **Step 3: Render the analytics section**

In the same file, insert this block immediately AFTER the closing `</div>` of the existing KPI grid (the `<div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4"> … </div>` that ends right before the `<div>` containing the "Kiosks in this store" heading):

```tsx
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        <KpiCard
          label="Receipts this month"
          value={formatNumber(analytics.monthTrend.current)}
          delta={analytics.monthTrend.pctChange ?? undefined}
          hint="vs last month"
          icon={TrendingUp}
        />
        <KpiCard
          label="Revenue this month"
          value={formatCurrency(analytics.revenueThisMonth)}
          icon={Receipt}
        />
        <KpiCard
          label="Paper saved"
          value={`${analytics.eco.paperKg.toFixed(1)} kg`}
          hint="this month"
        />
        <KpiCard
          label="Busiest day"
          value={analytics.peak.busiestDowLabel ?? "—"}
          hint="last 90 days"
          icon={CalendarClock}
        />
        <KpiCard
          label="Peak hour"
          value={analytics.peak.peakHourLabel ?? "—"}
          hint="last 90 days"
          icon={Clock}
        />
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Receipts over time</CardTitle>
          <CardDescription>Daily digital receipts, last 30 days</CardDescription>
        </CardHeader>
        <CardContent>
          <ReceiptsAreaChart data={analytics.daily} height={260} />
        </CardContent>
      </Card>
```

(The file already imports `KpiCard`, `formatNumber`, `Card`, `CardHeader`, `CardTitle`, `CardDescription`, `CardContent`.)

- [ ] **Step 4: Typecheck + build**

Run: `npx tsc --noEmit` (expect no errors)
Run: `npm run build` (expect success)

- [ ] **Step 5: Commit**

```bash
git add "app/(tenant)/tenant/stores/[storeId]/page.tsx"
git commit -m "feat(tenant): per-store analytics section on the store page"
```

---

## Task 7: Analytics page + nav

**Files:**
- Create: `app/(tenant)/tenant/analytics/page.tsx`
- Modify: `lib/nav.ts`

- [ ] **Step 1: Add the nav item**

In `lib/nav.ts`, add `LineChart` to the existing lucide import, then add the entry to `TENANT_NAV` immediately after the "Reports" line:

```ts
  { label: "Analytics", href: "/tenant/analytics", icon: LineChart },
```

(Add `LineChart` to the `import { ... } from "lucide-react";` at the top of `lib/nav.ts`.)

- [ ] **Step 2: Create the page**

Create `app/(tenant)/tenant/analytics/page.tsx`:

```tsx
import { PageHeader } from "@/components/page-header";
import { ExportButton } from "@/components/export-button";
import { BreakdownBarChart, StoreCompareChart } from "@/components/charts";
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { getStoresAnalytics } from "@/lib/data";
import { requireTenant } from "@/lib/session";
import { formatCurrency, formatNumber } from "@/lib/format";

export default async function AnalyticsPage() {
  const { organizationId } = await requireTenant();
  const { rows, monthlyByStore } = await getStoresAnalytics(organizationId);

  const byStore = rows.map((r) => ({ label: r.storeName, value: r.receiptsThisMonth }));
  const exportHeaders = ["Store", "Receipts (this month)", "Trend %", "Revenue (USD)", "Paper saved (kg)"];
  const exportRows = rows.map((r) => [
    r.storeName,
    r.receiptsThisMonth,
    r.trend.pctChange === null ? "—" : r.trend.pctChange,
    r.revenueThisMonth.toFixed(2),
    r.eco.paperKg.toFixed(1),
  ]);

  return (
    <>
      <PageHeader
        title="Analytics"
        description="Compare receipt volume, trends, and revenue across your stores."
      >
        <ExportButton
          label="Export analytics"
          filename="store-analytics.csv"
          headers={exportHeaders}
          rows={exportRows}
        />
      </PageHeader>

      {rows.length === 0 ? (
        <Card>
          <CardContent className="flex flex-col items-center justify-center gap-2 py-16 text-center">
            <p className="text-sm font-medium">No store data yet</p>
            <p className="max-w-xs text-xs text-muted-foreground">
              Once your stores start issuing receipts, comparisons show up here.
            </p>
          </CardContent>
        </Card>
      ) : (
        <>
          <Card>
            <CardHeader>
              <CardTitle>Receipts by store</CardTitle>
              <CardDescription>This month, highest first</CardDescription>
            </CardHeader>
            <CardContent>
              <BreakdownBarChart data={byStore} />
            </CardContent>
          </Card>

          <Card>
            <CardHeader>
              <CardTitle>Store comparison</CardTitle>
              <CardDescription>This month vs last, per store</CardDescription>
            </CardHeader>
            <CardContent className="divide-y">
              {rows.map((r) => (
                <div key={r.storeId} className="flex items-center justify-between py-3">
                  <div>
                    <p className="text-sm font-medium">{r.storeName}</p>
                    <p className="text-xs text-muted-foreground">
                      {formatNumber(r.receiptsThisMonth)} receipts · {formatCurrency(r.revenueThisMonth)}
                    </p>
                  </div>
                  <span
                    className={
                      r.trend.pctChange === null
                        ? "text-xs text-muted-foreground"
                        : r.trend.pctChange >= 0
                          ? "text-xs font-medium text-status-online"
                          : "text-xs font-medium text-destructive"
                    }
                  >
                    {r.trend.pctChange === null ? "new" : `${r.trend.pctChange >= 0 ? "▲" : "▼"} ${Math.abs(r.trend.pctChange)}%`}
                  </span>
                </div>
              ))}
            </CardContent>
          </Card>

          <Card>
            <CardHeader>
              <CardTitle>Trajectories</CardTitle>
              <CardDescription>Monthly receipts per store, last 9 months</CardDescription>
            </CardHeader>
            <CardContent>
              <StoreCompareChart data={monthlyByStore} />
            </CardContent>
          </Card>
        </>
      )}
    </>
  );
}
```

- [ ] **Step 3: Typecheck + build**

Run: `npx tsc --noEmit` (expect no errors)
Run: `npm run build` (expect success; `/tenant/analytics` appears in the route list)

- [ ] **Step 4: Commit**

```bash
git add "app/(tenant)/tenant/analytics/page.tsx" lib/nav.ts
git commit -m "feat(tenant): /tenant/analytics cross-store comparison page + nav"
```

---

## Task 8: Full verification

**Files:** none (verification only)

- [ ] **Step 1: Full test suite**

Run: `npm run test`
Expected: PASS — existing suites plus `lib/analytics.test.ts`.

- [ ] **Step 2: Typecheck**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Production build**

Run: `npm run build`
Expected: SUCCEEDS; both `/tenant/analytics` and `/tenant/stores/[storeId]` build.

- [ ] **Step 4: Commit any stragglers**

```bash
git status
# only if needed:
git add -A && git commit -m "chore: per-store analytics cleanup"
```

---

## Self-Review Notes

- **Spec coverage:** pure helpers (Tasks 1–2) cover series/trend/peak/comparison + all tested edge cases; `getStoreAnalytics` (Task 3) + `getStoresAnalytics` (Task 4) are the hybrid SQL layer; `StoreCompareChart` (Task 5); store-page deep view (Task 6); `/tenant/analytics` + nav (Task 7); verification (Task 8). Metrics — receipts/trend/revenue/eco/busiest-day/peak-hour — all present.
- **UTC consistency:** `dayKeys`/`monthKeys` and the SQL `to_char(date_trunc(...))` both produce `YYYY-MM-DD` / `YYYY-MM` in UTC, so the join in `bucketsToSeries` lines up. Documented caveat in the spec.
- **Naming consistency:** `StoreAnalytics`, `StoreComparisonRow`, `Trend`, `Peak`, `bucketsToSeries`, `dayKeys`, `monthKeys`, `computeTrend`, `buildPeak`, `toComparisonRows`, `StoreCompareChart`, `getStoreAnalytics`, `getStoresAnalytics` are used identically across tasks.
- **Out of scope (unchanged):** selectable ranges, hourly heatmaps, admin cross-tenant analytics, Stripe-actuals revenue, per-tenant timezone.
