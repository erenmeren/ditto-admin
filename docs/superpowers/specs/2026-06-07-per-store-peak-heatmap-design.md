# Per-store peak-hours heatmap — design

**Date:** 2026-06-07
**Status:** Approved (brainstorm) → ready for implementation plan
**Area:** Tenant analytics

## Summary

The store detail page already computes a 90-day "peak" — busiest day-of-week and
peak hour — but renders it as just two KPI labels, discarding the rest of the
distribution. This feature surfaces the full distribution as a **day-of-week × hour
heatmap** (7 × 24 grid) on the store detail page, computed in each store's **local
timezone**.

To make local-time meaningful, stores gain a `timezone` field (none exists today),
set via the store-create flow and a new **Edit store** dialog. Localizing the hour
axis also fixes a latent correctness issue: the existing peak-hour KPI is currently
computed in UTC, so a 7am local rush can display as the wrong hour.

## Motivation

- The dow × hour data is *already* aggregated for the peak KPIs; we throw ~166 of
  168 cells away. A heatmap is high value for low marginal cost.
- "Peak hour" is only actionable to a store owner in **local** time. UTC bucketing
  (shipped in the UTC-unification refactor) makes the current KPI misleading for any
  non-UTC store.
- Stores currently cannot be edited after creation at all — adding an Edit dialog
  closes that gap while giving timezone a natural home.

## Non-goals (v1)

- No tenant-wide aggregate heatmap (would mix stores across different zones — the
  hour axis stops being meaningful). Per-store only.
- No heatmap CSV export (the existing per-store/cross-store CSV exports are unchanged).
- No admin-side (cross-tenant) heatmap.
- No automatic timezone detection/geocoding from the store address — timezone is
  chosen explicitly from a curated list.

## Decisions (resolved during brainstorming)

| Decision | Choice | Rationale |
|---|---|---|
| Visualization | Day-of-week × hour heatmap (2-D grid) | The chosen direction; the data already exists. |
| Timezone model | **Per-store** `timezone` column | Most correct; supports multi-region chains. Tenant-wide was the alternative. |
| TZ editing UX | **Full Edit-store dialog** (name/address/timezone) | Also fixes the missing store-edit capability. |
| Time window | Last 90 days | Matches the existing peak computation. |
| Color | Emerald `var(--chart-1)` intensity scale | Consistent with existing charts. Brand color stays data-only (CLAUDE.md). |

## Data model

Add to `store` in `lib/db/schema.ts`:

```ts
timezone: text("timezone").notNull().default("UTC"),  // IANA name, e.g. America/New_York
```

- Migration generated via `npm run db:generate` → `lib/db/migrations/0010_*.sql`,
  applied with `npm run db:migrate`. The `NOT NULL DEFAULT 'UTC'` backfills the 3
  existing Roastwell stores to UTC (owner can then correct each via the Edit dialog).
- The `Store` view-model type (`lib/types.ts`) gains `timezone: string`; `getStore`
  / `getTenant` / `loadOrg` selects include it. Seed data (`db:seed`) sets a real
  zone (e.g. `America/Los_Angeles`) so the heatmap demo is sensible.

### Curated timezone list

A small shared constant (e.g. `lib/timezones.ts`) exporting ~12–20 common IANA zones
with friendly labels (US zones, UTC, London, a few EU/APAC). Used by both the
add-store and edit-store `<select>`s. Validated server-side against the same list so
a hand-crafted POST can't store a garbage zone (which would make `AT TIME ZONE`
throw at query time).

## Data layer

### Query (`getStoreAnalytics` in `lib/data.ts`)

Replace the two separate 1-D queries (`dowRows`, `hourRows`) with **one** 2-D query
over the last 90 days, localized to the store's timezone:

```sql
-- created_at is `timestamp` (without tz) storing UTC wall-clock, so it must be
-- re-anchored to UTC before converting to local — the double AT TIME ZONE is required.
extract(dow  from ((created_at AT TIME ZONE 'UTC') AT TIME ZONE :tz))::int   -- 0..6 Sun..Sat
extract(hour from ((created_at AT TIME ZONE 'UTC') AT TIME ZONE :tz))::int   -- 0..23
GROUP BY dow, hour
```

The store's timezone is available because `getStoreAnalytics` already loads the
store. The busiest-day and peak-hour KPIs are **derived from this single grid**
(sum across hours → busiest dow; sum across days → peak hour), so KPIs and heatmap
share one source of truth and are both in local time. `buildPeak` is reimplemented
on top of the grid via `buildHeatmap`, which returns the derived `peak` directly.

### Pure transform (`lib/analytics.ts`)

```ts
export interface Heatmap {
  grid: number[][];   // [7][24], grid[dow][hour] = receipt count
  max: number;        // max single-cell count (0 when empty) — drives intensity scale
  total: number;
  peak: Peak;         // busiest dow + peak hour derived from the grid
}

export function buildHeatmap(rows: { dow: number; hour: number; count: number }[]): Heatmap
```

Pure and IO-free, like the rest of `lib/analytics.ts`. `StoreAnalytics` **gains
`heatmap: Heatmap`** and **keeps `peak: Peak`** (now derived from the grid rather
than from separate queries), so the existing store-detail KPI cards that read
`analytics.peak` need no change. The heatmap card reads `analytics.heatmap`.

## UI

### `PeakHeatmap` client component (`components/`)

- A CSS-grid heatmap: 7 rows (Mon–Sun ordering for readability; the Postgres
  Sun=0 index is remapped) × 24 columns.
- Cell background = `var(--chart-1)` with opacity scaled by `count / max`
  (zero cells get a faint `--muted` tint, not pure emerald). A small fixed set of
  opacity buckets keeps it legible.
- Hover/`title`: e.g. "Tuesdays 8–9am · 23 receipts" (reuse `dowLabel` / `hourLabel`).
- Axis labels: full day names (or 3-letter) on rows; **sparse** hour ticks
  (12a / 6a / 12p / 6p) to avoid crowding 24 columns on mobile.
- All-zero empty state ("No receipts in the last 90 days yet").
- Caption: "Local time — {IANA zone}". Container-query / responsive so 24 columns
  remain usable on narrow screens (cells shrink; labels stay sparse).

### Store detail page (`app/(tenant)/tenant/stores/[storeId]/page.tsx`)

- Keep the two peak KPI cards (now local-time) as the at-a-glance headline.
- Add a `Card` titled "Busiest times" with `PeakHeatmap` below the existing
  "Receipts over time" chart.
- Add an **Edit store** trigger in the header (owner/admin only), next to the
  existing Claim-kiosk action.

### Store CRUD

- **`add-store-dialog.tsx`**: add a timezone `<select>` (curated list, sensible
  default). Wired through the existing `createStore` action.
- **`edit-store-dialog.tsx`** (new): pre-filled name / address / timezone.
  Reachable from the stores list rows and the detail-page header.
- **`lib/actions/stores.ts`**:
  - `createStore` / `createStoreForOrg` accept and persist `timezone` (validated
    against the curated list; fallback `'UTC'`).
  - New `updateStore(formData)` — `requireTenant`, owner/admin role check, verify
    the store belongs to the active org, update name/address/timezone, validate tz,
    `recordAudit(AUDIT.storeUpdated)` (new audit action constant), `revalidatePath`
    for `/tenant/stores`, the detail page, and `/tenant`.

## Authorization

- Viewing the heatmap: any tenant member of the owning org (same gate as the rest of
  the store detail page).
- Editing a store / its timezone: owner/admin only, enforced server-side in
  `updateStore` (mirrors `createStore`). The Edit trigger is hidden for members.

## Error handling

- Invalid/unknown timezone string never reaches SQL: validated against the curated
  list in the server actions; on mismatch fall back to `'UTC'` or reject.
- `getStoreAnalytics` keeps its current behavior (returns `null` when the store is
  missing); the 2-D query is a drop-in for the two it replaces.
- `getStoresAnalytics` and the cross-store comparison are **unaffected** (they
  aggregate monthly volume, not hour-of-day) and stay UTC/month-based.

## Testing

- `lib/analytics.test.ts`:
  - `buildHeatmap`: empty rows → all-zero grid, `max:0`, null peak; single cell;
    multiple cells with a clear max; tie handling for derived peak (first-wins,
    matching `pickPeakDow`/`pickPeakHour`); dow/hour out-of-range ignored.
  - Derived peak matches the old `buildPeak` for equivalent inputs (regression guard).
- `lib/actions` store test (following existing action-test patterns): `updateStore`
  rejects a non-owner/admin and rejects a store outside the active org.
- Timezone-list validation: an out-of-list zone is rejected / coerced.
- Manual / seed verification: `db:seed` store with a known zone renders a plausible
  local-time heatmap; confirm a receipt created at a known UTC instant lands in the
  expected local cell.

## Rollout

1. Schema + migration `0010`, regenerate types, update seed.
2. Pure `buildHeatmap` + tests (red→green).
3. `getStoreAnalytics` switches to the 2-D localized query.
4. `PeakHeatmap` component.
5. Store CRUD: timezone on create + new Edit dialog/action + audit constant.
6. Wire heatmap + Edit trigger into the store detail page.
7. `npm run build` + tests; manual check against seeded data.

No external accounts or env keys required — this is entirely account-free
(unlike Phase 0 activation work).
