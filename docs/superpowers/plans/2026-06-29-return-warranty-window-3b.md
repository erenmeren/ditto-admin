# Return & Warranty Window (Phase 3B) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show customers a computed return deadline and warranty expiry on the public document page, driven by two new tenant-wide settings.

**Architecture:** Mirror the Phase 3A support-contact slice end to end — pure helper (`lib/branding/coverage.ts`) for validation + date math, two nullable `tenantSettings` columns, a server action + card form on the branding page, and a render block on `/d/{token}`. All display math is `document.createdAt` + configured window; no new data pipeline, no firmware changes.

**Tech Stack:** Next.js 16 App Router (React 19, TS strict), Drizzle ORM over neon-http, Vitest, shadcn/ui (radix-nova), lucide-react.

## Global Constraints

- **Money/numbers:** windows are plain integers. `returnWindowDays` valid range `1..3650`; `warrantyPeriodMonths` valid range `1..120`. `null` = off.
- **Tenant-wide only.** Both values live on `tenantSettings` (PK = `organizationId`). No per-store override.
- **Migration churn:** after `npm run db:generate`, strip the generated `.sql` to contain ONLY the two `ADD COLUMN` statements (drizzle snapshot drift is known to emit spurious FK/index churn — see `memory/drizzle-snapshot-drift.md`).
- **Fail-soft on the public page:** invalid/`null` config must render as "window off", never throw. Same posture as logo/support.
- **Role gate:** only `owner`/`admin` may save settings (mirror `support-actions.ts`).
- **Audit:** reuse `AUDIT.brandingUpdated`; do not add a new audit action.
- **Tests stay green:** existing suite (305 tests) must pass; `npx tsc --noEmit` clean; `npm run build` succeeds.
- **Date formatting:** return shows day precision (`Jul 28, 2026`), warranty shows month + year (`Jun 2027`), via `toLocaleDateString("en-US", …)`.
- **Warranty month math:** add calendar months by copying the date and `setMonth(getMonth() + months)`, then clamp month-overflow to the last day of the target month (Jan 31 + 1 month → Feb 28/29, never rolling into March).

---

## File Structure

- `lib/db/schema.ts` — add 2 columns to `tenantSettings` (Task 1).
- `drizzle/0026_*.sql` — generated migration, stripped (Task 1).
- `lib/branding/coverage.ts` — new pure helper (Task 2).
- `lib/branding/coverage.test.ts` — new unit tests (Task 2).
- `lib/documents.ts` — `PublicDocument` + `getDocumentByToken` carry the 2 raw fields (Task 3).
- `lib/data.ts` — `TenantBranding` + `getTenantBranding` carry the 2 raw fields (Task 3).
- `app/(public)/d/[token]/page.tsx` — render the coverage block (Task 4).
- `app/(tenant)/tenant/branding/coverage-actions.ts` — new server action (Task 5).
- `components/coverage-window-form.tsx` — new card form (Task 5).
- `app/(tenant)/tenant/branding/page.tsx` — mount the form (Task 5).

---

### Task 1: Schema columns + migration

**Files:**
- Modify: `lib/db/schema.ts` (the `tenantSettings` table, near the `supportEmail`/`supportUrl` columns)
- Create: `drizzle/0026_*.sql` (generated, then stripped)

**Interfaces:**
- Produces: `tenantSettings.returnWindowDays` (`integer`, nullable) and `tenantSettings.warrantyPeriodMonths` (`integer`, nullable).

- [ ] **Step 1: Add the two columns to the schema**

In `lib/db/schema.ts`, find the support columns inside `tenantSettings`:

```ts
  // Optional customer-facing support contact, shown on the public /d/{token} page.
  supportEmail: text("support_email"),
  supportUrl: text("support_url"),
```

Add directly below them:

```ts
  // Optional time-based coverage windows shown on the public /d/{token} page.
  // Both null = off. Return deadline = document.createdAt + returnWindowDays days;
  // warranty expiry = document.createdAt + warrantyPeriodMonths calendar months.
  returnWindowDays: integer("return_window_days"),
  warrantyPeriodMonths: integer("warranty_period_months"),
```

(`integer` is already imported in this file — confirm the import line includes it.)

- [ ] **Step 2: Generate the migration**

Run: `npm run db:generate`
Expected: a new file `drizzle/0026_*.sql` is created.

- [ ] **Step 3: Strip the migration to only the new columns**

Open the new `drizzle/0026_*.sql`. Overwrite its entire contents with exactly:

```sql
ALTER TABLE "tenant_settings" ADD COLUMN "return_window_days" integer;--> statement-breakpoint
ALTER TABLE "tenant_settings" ADD COLUMN "warranty_period_months" integer;
```

Delete any other statements the generator added (FK re-creation, index churn). Leave the generated `drizzle/meta/` snapshot files as generated — only the `.sql` is hand-trimmed.

- [ ] **Step 4: Type-check**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add lib/db/schema.ts drizzle/
git commit -m "feat(3b): tenant_settings return_window_days + warranty_period_months columns"
```

---

### Task 2: Pure coverage helper (TDD)

**Files:**
- Create: `lib/branding/coverage.ts`
- Test: `lib/branding/coverage.test.ts`

**Interfaces:**
- Produces:
  - `isValidWindowDays(n: number): boolean` — integer in `1..3650`.
  - `isValidWarrantyMonths(n: number): boolean` — integer in `1..120`.
  - `interface CoverageWindow { untilDate: Date; expired: boolean }`
  - `interface Coverage { return: CoverageWindow | null; warranty: CoverageWindow | null; show: boolean }`
  - `coverageStatus(input: { createdAt: Date; returnWindowDays: number | null; warrantyPeriodMonths: number | null }, now: Date): Coverage`
  - `addCalendarMonths(date: Date, months: number): Date` (exported for testing the clamp edge)

- [ ] **Step 1: Write the failing tests**

Create `lib/branding/coverage.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import {
  isValidWindowDays,
  isValidWarrantyMonths,
  addCalendarMonths,
  coverageStatus,
} from "./coverage";

describe("isValidWindowDays", () => {
  it("accepts integers in 1..3650", () => {
    expect(isValidWindowDays(1)).toBe(true);
    expect(isValidWindowDays(30)).toBe(true);
    expect(isValidWindowDays(3650)).toBe(true);
  });
  it("rejects zero, negatives, non-integers, and over-max", () => {
    expect(isValidWindowDays(0)).toBe(false);
    expect(isValidWindowDays(-5)).toBe(false);
    expect(isValidWindowDays(1.5)).toBe(false);
    expect(isValidWindowDays(3651)).toBe(false);
    expect(isValidWindowDays(Number.NaN)).toBe(false);
  });
});

describe("isValidWarrantyMonths", () => {
  it("accepts integers in 1..120", () => {
    expect(isValidWarrantyMonths(1)).toBe(true);
    expect(isValidWarrantyMonths(12)).toBe(true);
    expect(isValidWarrantyMonths(120)).toBe(true);
  });
  it("rejects zero, negatives, non-integers, and over-max", () => {
    expect(isValidWarrantyMonths(0)).toBe(false);
    expect(isValidWarrantyMonths(-1)).toBe(false);
    expect(isValidWarrantyMonths(2.2)).toBe(false);
    expect(isValidWarrantyMonths(121)).toBe(false);
  });
});

describe("addCalendarMonths", () => {
  it("adds whole months", () => {
    expect(addCalendarMonths(new Date("2026-06-15T00:00:00Z"), 12).toISOString())
      .toBe("2027-06-15T00:00:00.000Z");
  });
  it("clamps month-overflow to the last day of the target month", () => {
    // Jan 31 + 1 month → Feb 28 (2026 is not a leap year), not Mar 3.
    expect(addCalendarMonths(new Date("2026-01-31T00:00:00Z"), 1).toISOString())
      .toBe("2026-02-28T00:00:00.000Z");
    // Leap year: Jan 31 + 1 month → Feb 29.
    expect(addCalendarMonths(new Date("2028-01-31T00:00:00Z"), 1).toISOString())
      .toBe("2028-02-29T00:00:00.000Z");
  });
});

describe("coverageStatus", () => {
  const createdAt = new Date("2026-06-01T00:00:00Z");

  it("returns show:false when both windows are off", () => {
    const c = coverageStatus(
      { createdAt, returnWindowDays: null, warrantyPeriodMonths: null },
      new Date("2026-06-10T00:00:00Z"),
    );
    expect(c.show).toBe(false);
    expect(c.return).toBeNull();
    expect(c.warranty).toBeNull();
  });

  it("treats invalid config as off", () => {
    const c = coverageStatus(
      { createdAt, returnWindowDays: 0, warrantyPeriodMonths: -3 },
      new Date("2026-06-10T00:00:00Z"),
    );
    expect(c.show).toBe(false);
  });

  it("computes an open return window", () => {
    const c = coverageStatus(
      { createdAt, returnWindowDays: 30, warrantyPeriodMonths: null },
      new Date("2026-06-10T00:00:00Z"), // before Jul 1
    );
    expect(c.show).toBe(true);
    expect(c.return).not.toBeNull();
    expect(c.return!.expired).toBe(false);
    expect(c.return!.untilDate.toISOString()).toBe("2026-07-01T00:00:00.000Z");
  });

  it("marks a passed return window expired", () => {
    const c = coverageStatus(
      { createdAt, returnWindowDays: 30, warrantyPeriodMonths: null },
      new Date("2026-08-01T00:00:00Z"), // after Jul 1
    );
    expect(c.return!.expired).toBe(true);
  });

  it("computes warranty expiry in calendar months", () => {
    const c = coverageStatus(
      { createdAt, returnWindowDays: null, warrantyPeriodMonths: 12 },
      new Date("2026-06-10T00:00:00Z"),
    );
    expect(c.warranty).not.toBeNull();
    expect(c.warranty!.expired).toBe(false);
    expect(c.warranty!.untilDate.toISOString()).toBe("2027-06-01T00:00:00.000Z");
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run lib/branding/coverage.test.ts`
Expected: FAIL — module `./coverage` not found.

- [ ] **Step 3: Implement the helper**

Create `lib/branding/coverage.ts`:

```ts
// lib/branding/coverage.ts
// Pure: validate + compute the optional return/warranty windows shown on the
// public document page. Single source of truth for the settings-form validation
// and the public-page display math. No IO. `now` is injected for determinism.

const DAY_MS = 24 * 60 * 60 * 1000;

export function isValidWindowDays(n: number): boolean {
  return Number.isInteger(n) && n >= 1 && n <= 3650;
}

export function isValidWarrantyMonths(n: number): boolean {
  return Number.isInteger(n) && n >= 1 && n <= 120;
}

/** Add whole calendar months, clamping overflow to the last day of the target month. */
export function addCalendarMonths(date: Date, months: number): Date {
  const d = new Date(date.getTime());
  const day = d.getUTCDate();
  d.setUTCDate(1); // avoid roll-over while shifting the month
  d.setUTCMonth(d.getUTCMonth() + months);
  // Last valid day of the now-current month.
  const lastDay = new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth() + 1, 0)).getUTCDate();
  d.setUTCDate(Math.min(day, lastDay));
  return d;
}

export interface CoverageWindow {
  untilDate: Date;
  expired: boolean;
}

export interface Coverage {
  return: CoverageWindow | null;
  warranty: CoverageWindow | null;
  show: boolean;
}

export function coverageStatus(
  input: {
    createdAt: Date;
    returnWindowDays: number | null;
    warrantyPeriodMonths: number | null;
  },
  now: Date,
): Coverage {
  const ret =
    input.returnWindowDays != null && isValidWindowDays(input.returnWindowDays)
      ? windowFrom(new Date(input.createdAt.getTime() + input.returnWindowDays * DAY_MS), now)
      : null;

  const warranty =
    input.warrantyPeriodMonths != null && isValidWarrantyMonths(input.warrantyPeriodMonths)
      ? windowFrom(addCalendarMonths(input.createdAt, input.warrantyPeriodMonths), now)
      : null;

  return { return: ret, warranty, show: ret != null || warranty != null };
}

function windowFrom(untilDate: Date, now: Date): CoverageWindow {
  return { untilDate, expired: now.getTime() > untilDate.getTime() };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run lib/branding/coverage.test.ts`
Expected: PASS (all cases).

- [ ] **Step 5: Full suite + type-check**

Run: `npm run test && npx tsc --noEmit`
Expected: all tests pass, no type errors.

- [ ] **Step 6: Commit**

```bash
git add lib/branding/coverage.ts lib/branding/coverage.test.ts
git commit -m "feat(3b): pure coverage helper — return/warranty window math + validators"
```

---

### Task 3: Data plumbing (read the new fields)

**Files:**
- Modify: `lib/documents.ts` (`PublicDocument` interface + `getDocumentByToken`)
- Modify: `lib/data.ts` (`TenantBranding` interface + `getTenantBranding` return)

**Interfaces:**
- Consumes: `tenantSettings.returnWindowDays`, `tenantSettings.warrantyPeriodMonths` (Task 1).
- Produces:
  - `PublicDocument.returnWindowDays: number | null`, `PublicDocument.warrantyPeriodMonths: number | null`.
  - `TenantBranding.returnWindowDays: number | null`, `TenantBranding.warrantyPeriodMonths: number | null`.

- [ ] **Step 1: Extend `PublicDocument` and its query in `lib/documents.ts`**

In the `PublicDocument` interface, after `supportUrl: string | null;` add:

```ts
  /** Tenant return window in days; null = off. */
  returnWindowDays: number | null;
  /** Tenant warranty period in months; null = off. */
  warrantyPeriodMonths: number | null;
```

In `getDocumentByToken`, extend the `.select({ … })` object (after `supportUrl: tenantSettings.supportUrl,`):

```ts
      returnWindowDays: tenantSettings.returnWindowDays,
      warrantyPeriodMonths: tenantSettings.warrantyPeriodMonths,
```

In the returned object, after `supportUrl: row.supportUrl ?? null,` add:

```ts
    returnWindowDays: row.returnWindowDays ?? null,
    warrantyPeriodMonths: row.warrantyPeriodMonths ?? null,
```

- [ ] **Step 2: Extend `TenantBranding` and `getTenantBranding` in `lib/data.ts`**

In the `TenantBranding` interface, after `supportUrl: string | null;` add:

```ts
  returnWindowDays: number | null;
  warrantyPeriodMonths: number | null;
```

In the `return { … }` of `getTenantBranding`, after `supportUrl: s?.supportUrl ?? null,` add:

```ts
    returnWindowDays: s?.returnWindowDays ?? null,
    warrantyPeriodMonths: s?.warrantyPeriodMonths ?? null,
```

- [ ] **Step 3: Type-check + build (fields are additive — nothing consumes them yet)**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors; build succeeds.

- [ ] **Step 4: Full suite**

Run: `npm run test`
Expected: all tests pass (no behavioral change yet).

- [ ] **Step 5: Commit**

```bash
git add lib/documents.ts lib/data.ts
git commit -m "feat(3b): carry return/warranty window config through public-doc + branding view-models"
```

---

### Task 4: Render the coverage block on the public page

**Files:**
- Modify: `app/(public)/d/[token]/page.tsx`

**Interfaces:**
- Consumes: `coverageStatus` (Task 2); `PublicDocument.returnWindowDays`/`warrantyPeriodMonths`, `document.createdAt`, `document.brandColor` (Task 3 / existing).

- [ ] **Step 1: Import the helper and lucide icons**

At the top of `app/(public)/d/[token]/page.tsx`, add `RotateCcw` and `ShieldCheck` to the existing lucide import:

```ts
import { Check, Download, Leaf, FileText, SearchX, Mail, ExternalLink, RotateCcw, ShieldCheck } from "lucide-react";
```

Add the helper import near the other `@/lib` imports:

```ts
import { coverageStatus } from "@/lib/branding/coverage";
```

- [ ] **Step 2: Compute coverage in the ready branch**

In the ready (download) branch, just after `const support = supportLinks(document);`, add:

```ts
  const coverage = coverageStatus(document, new Date());
```

- [ ] **Step 3: Render the coverage block**

Immediately after the closing `)}` of the `{support.show && ( … )}` block (before the final `</Shell>`), insert:

```tsx
      {coverage.show && (
        <div className="space-y-2 border-t px-6 py-4 text-center text-xs">
          {coverage.return && (
            <p className="flex items-center justify-center gap-1.5">
              <RotateCcw className="size-3.5" style={{ color: coverage.return.expired ? undefined : accent }} />
              {coverage.return.expired ? (
                <span className="text-muted-foreground">
                  Return period ended (was{" "}
                  {coverage.return.untilDate.toLocaleDateString("en-US", { dateStyle: "medium" })})
                </span>
              ) : (
                <span className="font-medium" style={{ color: accent }}>
                  Returns accepted until{" "}
                  {coverage.return.untilDate.toLocaleDateString("en-US", { dateStyle: "medium" })}
                </span>
              )}
            </p>
          )}
          {coverage.warranty && (
            <p className="flex items-center justify-center gap-1.5">
              <ShieldCheck className="size-3.5" style={{ color: coverage.warranty.expired ? undefined : accent }} />
              {coverage.warranty.expired ? (
                <span className="text-muted-foreground">
                  Warranty expired{" "}
                  {coverage.warranty.untilDate.toLocaleDateString("en-US", { year: "numeric", month: "short" })}
                </span>
              ) : (
                <span className="font-medium" style={{ color: accent }}>
                  Under warranty until{" "}
                  {coverage.warranty.untilDate.toLocaleDateString("en-US", { year: "numeric", month: "short" })}
                </span>
              )}
            </p>
          )}
        </div>
      )}
```

- [ ] **Step 4: Type-check + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no errors; build succeeds.

- [ ] **Step 5: Commit**

```bash
git add "app/(public)/d/[token]/page.tsx"
git commit -m "feat(3b): show return/warranty window on the public document page"
```

---

### Task 5: Settings action + card form

**Files:**
- Create: `app/(tenant)/tenant/branding/coverage-actions.ts`
- Create: `components/coverage-window-form.tsx`
- Modify: `app/(tenant)/tenant/branding/page.tsx`

**Interfaces:**
- Consumes: `isValidWindowDays`, `isValidWarrantyMonths` (Task 2); `TenantBranding.returnWindowDays`/`warrantyPeriodMonths` (Task 3).
- Produces: `saveCoverageWindow(formData: FormData): Promise<{ ok: boolean; error?: string }>`.

- [ ] **Step 1: Write the server action**

Create `app/(tenant)/tenant/branding/coverage-actions.ts`:

```ts
"use server";

// Persist the optional return/warranty windows (shown on /d/{token}).
import { revalidatePath } from "next/cache";
import { db } from "@/lib/db";
import { tenantSettings } from "@/lib/db/schema";
import { requireTenant } from "@/lib/session";
import { isValidWindowDays, isValidWarrantyMonths } from "@/lib/branding/coverage";
import { recordAudit, AUDIT } from "@/lib/audit";

export interface SaveCoverageResult {
  ok: boolean;
  error?: string;
}

/** Parse a blank-or-integer field. Returns: null (blank), number (parsed), or "invalid". */
function parseWindow(raw: string): number | null | "invalid" {
  const s = raw.trim();
  if (!s) return null;
  if (!/^\d+$/.test(s)) return "invalid";
  return Number(s);
}

export async function saveCoverageWindow(formData: FormData): Promise<SaveCoverageResult> {
  const { ctx, organizationId } = await requireTenant();
  const membership = ctx.organizations.find((o) => o.id === organizationId);
  if (!membership || !["owner", "admin"].includes(membership.role)) {
    return { ok: false, error: "You don't have permission to edit this." };
  }

  const days = parseWindow(String(formData.get("returnWindowDays") ?? ""));
  const months = parseWindow(String(formData.get("warrantyPeriodMonths") ?? ""));

  if (days === "invalid" || (typeof days === "number" && !isValidWindowDays(days))) {
    return { ok: false, error: "Return window must be a whole number of days (1–3650), or blank." };
  }
  if (months === "invalid" || (typeof months === "number" && !isValidWarrantyMonths(months))) {
    return { ok: false, error: "Warranty must be a whole number of months (1–120), or blank." };
  }

  const returnWindowDays = days as number | null;
  const warrantyPeriodMonths = months as number | null;

  await db
    .insert(tenantSettings)
    .values({ organizationId, returnWindowDays, warrantyPeriodMonths })
    .onConflictDoUpdate({
      target: tenantSettings.organizationId,
      set: { returnWindowDays, warrantyPeriodMonths, updatedAt: new Date() },
    });

  await recordAudit({
    organizationId,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.brandingUpdated,
  });
  revalidatePath("/tenant/branding");
  return { ok: true };
}
```

- [ ] **Step 2: Write the card form component**

Create `components/coverage-window-form.tsx`:

```tsx
"use client";

import * as React from "react";
import { toast } from "sonner";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { saveCoverageWindow } from "@/app/(tenant)/tenant/branding/coverage-actions";

export function CoverageWindowForm({
  initialReturnDays,
  initialWarrantyMonths,
  canEdit,
}: {
  initialReturnDays: number | null;
  initialWarrantyMonths: number | null;
  canEdit: boolean;
}) {
  const [pending, setPending] = React.useState(false);

  async function action(formData: FormData) {
    setPending(true);
    const res = await saveCoverageWindow(formData);
    setPending(false);
    if (res.ok) toast.success("Return & warranty window saved");
    else toast.error("Couldn't save", { description: res.error });
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle className="text-base">Return &amp; warranty window</CardTitle>
        <CardDescription>
          Optional. Shown to customers on the document page as a return deadline and
          warranty expiry, counted from when the document was issued. Leave blank to hide.
        </CardDescription>
      </CardHeader>
      <CardContent>
        <form action={action} className="flex flex-col gap-3 sm:flex-row sm:items-end">
          <label className="flex-1 text-sm">
            <span className="mb-1 block text-muted-foreground">Return window (days)</span>
            <Input
              name="returnWindowDays"
              type="number"
              min={1}
              max={3650}
              defaultValue={initialReturnDays ?? ""}
              placeholder="30"
              disabled={!canEdit}
            />
          </label>
          <label className="flex-1 text-sm">
            <span className="mb-1 block text-muted-foreground">Warranty (months)</span>
            <Input
              name="warrantyPeriodMonths"
              type="number"
              min={1}
              max={120}
              defaultValue={initialWarrantyMonths ?? ""}
              placeholder="12"
              disabled={!canEdit}
            />
          </label>
          {canEdit && (
            <Button type="submit" disabled={pending}>
              {pending ? "Saving…" : "Save"}
            </Button>
          )}
        </form>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 3: Mount the form on the branding page**

In `app/(tenant)/tenant/branding/page.tsx`, add the import next to the support-form import:

```ts
import { CoverageWindowForm } from "@/components/coverage-window-form";
```

After the existing `<div className="mt-6"> … <SupportContactForm … /> … </div>` block, add a sibling block:

```tsx
      <div className="mt-6">
        <CoverageWindowForm
          initialReturnDays={branding.returnWindowDays}
          initialWarrantyMonths={branding.warrantyPeriodMonths}
          canEdit={canEdit}
        />
      </div>
```

- [ ] **Step 4: Type-check + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no errors; build succeeds.

- [ ] **Step 5: Full suite**

Run: `npm run test`
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add "app/(tenant)/tenant/branding/coverage-actions.ts" components/coverage-window-form.tsx "app/(tenant)/tenant/branding/page.tsx"
git commit -m "feat(3b): return & warranty window settings card + save action"
```

---

## Self-Review

**Spec coverage:**
- Schema (2 nullable cols, stripped migration) → Task 1. ✓
- Pure helper (validators + `coverageStatus` + calendar-month clamp, injected `now`) → Task 2. ✓
- Data plumbing (`PublicDocument` + `TenantBranding`) → Task 3. ✓
- Public-page render (open/expired wording, accent vs muted, day vs month precision) → Task 4. ✓
- Settings card + action (role gate, validate, upsert, audit, revalidate) → Task 5. ✓
- Testing (pure deterministic helper tests; suite/tsc/build green) → Task 2 + per-task verification. ✓
- Non-goals (no per-store, no capture, no email, no firmware) → respected; nothing in any task adds them. ✓

**Placeholder scan:** none — every code step shows complete code; no TBD/TODO.

**Type consistency:** `coverageStatus(input, now)` and `Coverage`/`CoverageWindow` shapes are identical in Task 2 (definition), Task 4 (consumer); validator names `isValidWindowDays`/`isValidWarrantyMonths` match across Task 2 and Task 5; field names `returnWindowDays`/`warrantyPeriodMonths` are identical across schema (Task 1), view-models (Task 3), render (Task 4), form/action (Task 5). ✓
