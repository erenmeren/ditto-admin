# Return & Warranty Window (Phase 3B) — Design

**Date:** 2026-06-29
**Status:** Approved, ready for planning
**Builds on:** Phase 3A (branded public document page, support-contact settings)

## Problem

A customer scans the QR on a printed document and lands on `/d/{token}`. Today the
page shows the document image, provenance, and optional support links. It does not
tell the customer the two things they most often need after a purchase: **how long
they have to return** the item and **how long it is under warranty**.

A literal "warranty lookup" (type a serial/order number) is not feasible: a
`document` row stores a rendered image plus technical render metadata only — there
is **no parsed purchase data** (no product, price, order id), and the capability
token is already the customer's only key. There is nothing to look up *by*.

What we *do* have on every document is `createdAt`. Combined with a tenant-level
policy, that is enough to compute and display a return deadline and a warranty
expiry. This feature adds exactly that.

## Goal

Let a tenant configure a return window (in days) and a warranty period (in months).
On the public document page, compute and display the resulting dates from the
document's `createdAt`. No new data pipeline, no printer/firmware changes.

## Non-goals

- No per-store override — windows are tenant-wide (like brand color, support links).
  YAGNI until stores demonstrably need different policies.
- No customer data capture / warranty registration (deferred Phase 3 candidate B).
- No parsing of document contents. Math is purely `createdAt` + configured window.
- No email/notification. Display only.

## Architecture

Mirrors the Phase 3A support-contact slice end to end: pure helper → server action →
card form → public-page block.

### 1. Schema (`lib/db/schema.ts`, `tenantSettings`)

Two new nullable columns, both `null` = "off / not configured":

- `returnWindowDays` — `integer("return_window_days")` (nullable)
- `warrantyPeriodMonths` — `integer("warranty_period_months")` (nullable)

One additive migration containing **only** these two `ADD COLUMN` statements. Per the
known drizzle snapshot-drift gotcha, strip any spurious FK/index churn the generator
emits so the `.sql` file contains just this change.

### 2. Pure helper `lib/branding/coverage.ts`

No IO. Single source of truth for validation (settings form) and display math
(public page). Mirrors `lib/branding/support.ts`.

```ts
export function isValidWindowDays(n: number): boolean      // integer, 1..3650
export function isValidWarrantyMonths(n: number): boolean  // integer, 1..120

export interface CoverageWindow {
  untilDate: Date;
  expired: boolean;
}
export interface Coverage {
  return: CoverageWindow | null;    // null when returnWindowDays not set/invalid
  warranty: CoverageWindow | null;  // null when warrantyPeriodMonths not set/invalid
  show: boolean;                    // true when either window is present
}

export function coverageStatus(
  input: {
    createdAt: Date;
    returnWindowDays: number | null;
    warrantyPeriodMonths: number | null;
  },
  now: Date,
): Coverage
```

Date math:
- Return: `untilDate = createdAt + returnWindowDays days`. `expired = now > untilDate`.
- Warranty: `untilDate = createdAt + warrantyPeriodMonths` calendar months via a copy +
  `setMonth(getMonth() + months)`. Handle the month-overflow edge (e.g. Jan 31 + 1
  month) by clamping to the last day of the target month rather than rolling into the
  following month. `expired = now > untilDate`.
- Invalid/`null` config → that window is `null`. `show = return != null || warranty != null`.

### 3. Data plumbing

- `lib/documents.ts` — `PublicDocument` gains `returnWindowDays: number | null` and
  `warrantyPeriodMonths: number | null`. `getDocumentByToken` selects both from the
  `tenantSettings` join and passes them through. The page derives display via
  `coverageStatus(document, new Date())` (document already carries `createdAt`).
- `lib/data.ts` — the tenant branding view-model (read by the branding page) gains
  both raw fields so the settings form prefills with current values.

### 4. Settings UI (branding page)

A new sibling card **"Return & warranty window"** + server action
`saveCoverageWindow`, an exact mirror of `support-actions.ts`:

- `requireTenant()`, then owner/admin role gate (same check as support action).
- Read `returnWindowDays` / `warrantyPeriodMonths` from `FormData`. Blank → `null`.
  Non-blank must pass `isValidWindowDays` / `isValidWarrantyMonths`, else return a
  friendly `{ ok:false, error }`.
- Upsert into `tenantSettings` (`onConflictDoUpdate`, bump `updatedAt`).
- `recordAudit({ action: AUDIT.brandingUpdated, ... })` (reuse existing action).
- `revalidatePath("/tenant/branding")`.

New client component `components/coverage-window-form.tsx` mirroring
`support-contact-form.tsx`: two number inputs (return days, warranty months) with
helper text "Leave blank to hide", `canEdit` gating, sonner toast on save. Rendered
on the branding page next to the support-contact card.

### 5. Public page (`app/(public)/d/[token]/page.tsx`)

Add a coverage block below the support section, rendered only when
`coverage.show`. For each present window, a row with an icon + label:

- Return, open: `↩ Returns accepted until <Mon DD>` — accent-styled (uses `accent`).
- Return, expired: `↩ Return period ended (was <Mon DD>)` — muted.
- Warranty, open: `🛡 Under warranty until <Mon YYYY>` — accent-styled.
- Warranty, expired: `🛡 Warranty expired <Mon YYYY>` — muted.

Icons from lucide (`RotateCcw`, `ShieldCheck`). Date formatting via
`toLocaleDateString("en-US", ...)` consistent with the existing page (return shows
day precision; warranty shows month + year). Visual treatment matches the existing
card sections (bordered, `px-6 py-4`, small muted text).

## Error handling

- Invalid or out-of-range settings values are rejected at the action with a friendly
  message; nothing is persisted.
- The helper treats any `null`/invalid config defensively as "window off" so the
  public page can never throw on bad data — same fail-soft posture as logo/support.

## Testing

- New `lib/branding/coverage.test.ts` (pure, deterministic via injected `now`):
  - validation bounds (0, negative, non-integer, over-max → false; in-range → true)
  - return active vs expired around the boundary
  - warranty calendar-month math incl. Jan 31 + 1 month edge
  - both unset → `show:false`; one set → `show:true`, other side `null`
- Existing suite (305 tests) stays green. `tsc --noEmit` clean, `next build` succeeds.

## Rollout

Additive migration; safe to apply to prod with no downtime (same shape as the 3A
`support_email`/`support_url` columns). No env or firmware dependency.
