# Eco + Analytics removal — design

**Date:** 2026-07-15
**Status:** Approved (full eco purge confirmed by user)

## Goal

Remove the two eco cards from the tenant Reports page, delete the tenant
Analytics page entirely (UI + nav + backing data-layer code), and purge the
now-dead eco subsystem from the codebase.

## Context

- The Reports page's "Eco savings over time" chart and "Eco impact"
  (`EcoSavingsCard`) are the last UI consumers of `lib/eco.ts`, apart from one
  CSV column on the Analytics page — which is being deleted anyway.
- `getCustomerDetail` in `lib/data.ts` still computes an `eco` field that no UI
  reads (leftover from the earlier dashboard cleanup).
- The Analytics page's backend chain — `getStoresAnalytics` (`lib/data.ts`),
  `toComparisonRows` / `StoreComparisonRow` (`lib/analytics.ts`),
  `StoreCompareChart` (`components/charts.tsx`) — has no other consumers.
- After the eco chart goes, `MetricAreaChart` in `components/charts.tsx` is
  also orphaned.

## Changes

### 1. Reports page (`app/(tenant)/tenant/reports/page.tsx`)

- Remove the bottom `lg:grid-cols-3` grid: the "Eco savings over time" card
  (`MetricAreaChart`) and `EcoSavingsCard`.
- Remove the `ecoOverTime`, `totalActivations`, `eco` computations and the
  `@/lib/eco` / `@/components/eco-savings` imports.
- Update the page description ("Activations, breakdowns, and eco savings
  across your fleet.") to drop the eco mention, e.g. "Activations and
  breakdowns across your fleet."
- Keep: Activations over time, By store, By device cards, and the CSV export
  (eco was never in the export — unchanged).

### 2. Analytics page — full removal

- Delete `app/(tenant)/tenant/analytics/page.tsx`.
- `lib/nav.ts`: remove the "Analytics" nav entry and the `LineChart` icon
  import if it becomes unused.
- `lib/data.ts`: delete `getStoresAnalytics` and its return-type block; update
  the two comments that reference it (near lines 226 and 361).
- `lib/analytics.ts`: delete `toComparisonRows` and `StoreComparisonRow`, and
  the `./eco` import. **Keep** `computeTrend`, `dayKeys`, `monthKeys`,
  `bucketsToSeries`, `countByDayKey`, `countByMonthKey` — all used by
  `getStoreAnalytics` (store detail page, which stays).
- `components/charts.tsx`: delete `StoreCompareChart` and `MetricAreaChart`.

### 3. Eco purge

- Delete `lib/eco.ts` and `components/eco-savings.tsx`.
- `lib/data.ts`: remove the `eco` field from `CustomerDetail` and the
  `computeEcoSavings` import/call in `getCustomerDetail` (no UI reads it).

### 4. Tests & verification

- `lib/analytics.test.ts`: delete the `toComparisonRows` describe block; keep
  the `computeTrend` (and bucketing) tests.
- Gates: `npx tsc --noEmit`, `npx vitest run`, plus a grep sweep for leftover
  `eco` / `getStoresAnalytics` / `StoreCompareChart` / `MetricAreaChart`
  references.
- Per the batch-deploy workflow: code changes stay in the working tree — no
  code commit, no deploy. User tests locally (dev server on port 3000) and
  requests the batch commit later.

## Out of scope

- The store detail page's `getStoreAnalytics` and its charts.
- Admin pages (no eco/analytics UI remains there).
- No DB/schema changes — everything here is code-only.
