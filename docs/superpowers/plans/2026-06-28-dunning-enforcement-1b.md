# Phase 1B — Dunning & Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automate the delinquency lifecycle — generate last month's invoices, auto-send card tenants, sweep unpaid invoices to overdue, and hard-block ingest/API for orgs past a 7-day grace.

**Architecture:** One additive `invoice.dueDate` column (set on send); a pure `lib/billing/dunning.ts` holding all date/grace/block decisions; a self-healing daily `/api/cron/billing` that generates→auto-sends→sweeps; and a shared `isOrgPaymentBlocked` IO helper that refactors the existing inline suspension checks in ingest + the `/api/v1` guard and adds the past-due (402) block, plus a tenant banner/redirect.

**Tech Stack:** Next.js 16 route handlers + server components, Drizzle over Neon, `stripe` SDK (test mode), Vitest (pure functions only — no DB harness).

## Global Constraints

- **Auto-send only card tenants:** the cron pushes invoices for orgs with `tenantSettings.cardLast4` set (→ `charge_automatically` via 1A's `sendInvoiceToStripe`); no-card tenants stay `draft`.
- **Grace then block:** `GRACE_DAYS = 7`. Overdue → banner immediately; ingest/API hard-block only once `dueDate + 7 days < now`.
- **Block status codes:** `past_due` → **HTTP 402**; `suspended` → **HTTP 403** (unchanged behavior).
- **Self-healing cron:** always targets the *previous calendar month*; fully idempotent (generation only updates drafts, auto-send guards on `already_sent`, sweep only `sent → overdue`). Daily `0 9 * * *` (Vercel Hobby = daily only).
- **Enforcement fails open:** any DB error in `isOrgPaymentBlocked` returns `{ blocked:false }` + `reportError` — never block a paying customer on an infra blip.
- **Money is integer cents.** Credits code is untouched (no double-billing).
- **Prod-touching steps are deferred to the user** (`.env.local` is prod): `npm run db:migrate` and the live cron/enforcement smoke are acceptance steps the controller/user runs, NOT the subagents. Subagents stop at `db:generate` + build/tsc/unit-tests.
- Verification per task: `npm run test` (currently 259, stays green), `npm run build`, `npx tsc --noEmit`. Dev server is on **:3001**.

## File Map

| File | Responsibility | Task |
|---|---|---|
| `lib/db/schema.ts` (modify) + `drizzle/0024_*.sql` (generated) | `invoice.dueDate` column | 1 |
| `lib/billing/stripe-billing.ts` (modify) | set `dueDate` when persisting a sent invoice | 1 |
| `lib/billing/dunning.ts` (new) + `.test.ts` | pure: GRACE_DAYS, previousMonthMarker, isOverdue, isPastGrace, paymentBlockVerdict | 2 |
| `lib/billing/enforcement.ts` (new) | `isOrgPaymentBlocked` IO helper | 3 |
| `app/api/ingest/route.ts` (modify) | use the helper (402 past_due / 403 suspended) | 3 |
| `lib/api/guard.ts` (modify) | use the helper (402 / 403) | 3 |
| `lib/audit.ts` (modify) | `AUDIT.invoiceOverdue` | 4 |
| `lib/billing/billing-cron.ts` (new) | `runBillingCron(now)` | 4 |
| `app/api/cron/billing/route.ts` (new) + `vercel.json` (modify) | cron route + schedule | 4 |
| `app/(tenant)/layout.tsx` (modify) | overdue banner + blocked redirect | 5 |

---

### Task 1: `invoice.dueDate` column + populate it on send

**Files:**
- Modify: `lib/db/schema.ts` (invoice table, after `hostedInvoiceUrl`)
- Generate: `drizzle/0024_*.sql` (via `npm run db:generate`)
- Modify: `lib/billing/stripe-billing.ts` (the `sendInvoiceToStripe` persist block)

**Interfaces:**
- Produces: `invoice.dueDate` (Drizzle column, `Date | null`) — consumed by the sweep (Task 4) and enforcement (Task 3).

- [ ] **Step 1: Add the column to the schema**

In `lib/db/schema.ts`, the `invoice` table currently has:

```ts
    stripeInvoiceId: text("stripe_invoice_id"),
    hostedInvoiceUrl: text("hosted_invoice_url"),
    createdAt: timestamp("created_at")
```

Insert a `dueDate` column between `hostedInvoiceUrl` and `createdAt`:

```ts
    stripeInvoiceId: text("stripe_invoice_id"),
    hostedInvoiceUrl: text("hosted_invoice_url"),
    // Local mirror of the Stripe invoice due_date (send_invoice invoices only;
    // charge_automatically has none → null). Drives the overdue sweep + grace.
    dueDate: timestamp("due_date"),
    createdAt: timestamp("created_at")
```

- [ ] **Step 2: Generate the migration**

Run: `npm run db:generate`
Expected: a new `drizzle/0024_*.sql` containing `ALTER TABLE "invoice" ADD COLUMN "due_date" timestamp;` (additive, nullable — safe). Do NOT run `db:migrate` (that hits prod; it is a deferred deploy step).

- [ ] **Step 3: Populate `dueDate` when persisting a sent invoice**

In `lib/billing/stripe-billing.ts`, the `sendInvoiceToStripe` persist block currently reads:

```ts
  await db
    .update(invoiceTable)
    .set({
      stripeInvoiceId: finalized.id,
      hostedInvoiceUrl: finalized.hosted_invoice_url ?? null,
      status: "sent",
    })
    .where(eq(invoiceTable.id, inv.id));
```

Add the `dueDate` line:

```ts
  await db
    .update(invoiceTable)
    .set({
      stripeInvoiceId: finalized.id,
      hostedInvoiceUrl: finalized.hosted_invoice_url ?? null,
      dueDate: finalized.due_date != null ? new Date(finalized.due_date * 1000) : null,
      status: "sent",
    })
    .where(eq(invoiceTable.id, inv.id));
```

(`finalized.due_date` is a Unix-seconds number or null on the Stripe `Invoice`.)

- [ ] **Step 4: Typecheck + build + suite**

Run: `npx tsc --noEmit && npm run build && npm run test`
Expected: no type errors; build OK; 259 tests green (no test touches this path).

- [ ] **Step 5: Commit**

```bash
git add lib/db/schema.ts drizzle/ lib/billing/stripe-billing.ts
git commit -m "feat(billing): add invoice.dueDate column, populate on send (1B)"
```

---

### Task 2: Pure dunning decision helpers

**Files:**
- Create: `lib/billing/dunning.ts`
- Test: `lib/billing/dunning.test.ts`

**Interfaces:**
- Consumes: `isSuspended` from `lib/billing/billing-status.ts` (existing, pure: `(subscriptionStatus: string | null) => boolean`).
- Produces:
  - `GRACE_DAYS = 7`
  - `previousMonthMarker(now: Date): Date`
  - `isOverdue(inv: { status: string; dueDate: Date | null }, now: Date): boolean`
  - `isPastGrace(inv: { status: string; dueDate: Date | null }, now: Date): boolean`
  - `paymentBlockVerdict(input: { subscriptionStatus: string | null; hasPastGraceOverdue: boolean }): { blocked: boolean; reason: "suspended" | "past_due" | null }`

- [ ] **Step 1: Write the failing test**

Create `lib/billing/dunning.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import {
  GRACE_DAYS,
  previousMonthMarker,
  isOverdue,
  isPastGrace,
  paymentBlockVerdict,
} from "./dunning";

describe("previousMonthMarker", () => {
  it("returns a date inside the previous calendar month", () => {
    const m = previousMonthMarker(new Date(2026, 6, 3)); // Jul 3 2026
    expect(m.getFullYear()).toBe(2026);
    expect(m.getMonth()).toBe(5); // June
  });
  it("rolls over the year for January", () => {
    const m = previousMonthMarker(new Date(2026, 0, 10)); // Jan 10 2026
    expect(m.getFullYear()).toBe(2025);
    expect(m.getMonth()).toBe(11); // December
  });
});

describe("isOverdue", () => {
  const now = new Date("2026-07-01T00:00:00Z");
  it("is true for a sent invoice whose dueDate has passed", () => {
    expect(isOverdue({ status: "sent", dueDate: new Date("2026-06-30T00:00:00Z") }, now)).toBe(true);
  });
  it("is false when not yet due", () => {
    expect(isOverdue({ status: "sent", dueDate: new Date("2026-07-05T00:00:00Z") }, now)).toBe(false);
  });
  it("is false for non-sent or null-dueDate invoices", () => {
    expect(isOverdue({ status: "draft", dueDate: new Date("2026-06-01T00:00:00Z") }, now)).toBe(false);
    expect(isOverdue({ status: "paid", dueDate: new Date("2026-06-01T00:00:00Z") }, now)).toBe(false);
    expect(isOverdue({ status: "sent", dueDate: null }, now)).toBe(false);
  });
});

describe("isPastGrace", () => {
  const now = new Date("2026-07-20T00:00:00Z");
  it("is true when an overdue invoice is past dueDate + GRACE_DAYS", () => {
    // dueDate Jul 1 + 7d = Jul 8 < Jul 20
    expect(isPastGrace({ status: "overdue", dueDate: new Date("2026-07-01T00:00:00Z") }, now)).toBe(true);
  });
  it("is false while still within grace", () => {
    // dueDate Jul 15 + 7d = Jul 22 > Jul 20
    expect(isPastGrace({ status: "overdue", dueDate: new Date("2026-07-15T00:00:00Z") }, now)).toBe(false);
  });
  it("is false for a null dueDate (charge_automatically) or non-overdue", () => {
    expect(isPastGrace({ status: "overdue", dueDate: null }, now)).toBe(false);
    expect(isPastGrace({ status: "sent", dueDate: new Date("2026-07-01T00:00:00Z") }, now)).toBe(false);
  });
});

describe("paymentBlockVerdict", () => {
  it("blocks suspended subscriptions (takes precedence)", () => {
    expect(paymentBlockVerdict({ subscriptionStatus: "canceled", hasPastGraceOverdue: true }))
      .toEqual({ blocked: true, reason: "suspended" });
  });
  it("blocks past-grace overdue when not suspended", () => {
    expect(paymentBlockVerdict({ subscriptionStatus: "active", hasPastGraceOverdue: true }))
      .toEqual({ blocked: true, reason: "past_due" });
  });
  it("allows a clean org", () => {
    expect(paymentBlockVerdict({ subscriptionStatus: "active", hasPastGraceOverdue: false }))
      .toEqual({ blocked: false, reason: null });
    expect(paymentBlockVerdict({ subscriptionStatus: null, hasPastGraceOverdue: false }))
      .toEqual({ blocked: false, reason: null });
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run lib/billing/dunning.test.ts`
Expected: FAIL — `Cannot find module './dunning'`.

- [ ] **Step 3: Write the implementation**

Create `lib/billing/dunning.ts`:

```ts
// lib/billing/dunning.ts
// Pure, IO-free dunning decisions (Phase 1B). No DB/env/Stripe imports — the IO
// that uses these lives in enforcement.ts and billing-cron.ts.
import { isSuspended } from "./billing-status";

export const GRACE_DAYS = 7;
const DAY_MS = 86_400_000;

/** A stable mid-day date inside the calendar month BEFORE `now` (local time, to
 * match runInvoiceGeneration's local month boundaries). */
export function previousMonthMarker(now: Date): Date {
  return new Date(now.getFullYear(), now.getMonth() - 1, 15, 12, 0, 0, 0);
}

/** A sent invoice whose due date has passed should flip to overdue. */
export function isOverdue(
  inv: { status: string; dueDate: Date | null },
  now: Date,
): boolean {
  return inv.status === "sent" && inv.dueDate != null && inv.dueDate.getTime() < now.getTime();
}

/** An overdue invoice past the grace window should trigger a hard block. */
export function isPastGrace(
  inv: { status: string; dueDate: Date | null },
  now: Date,
): boolean {
  return (
    inv.status === "overdue" &&
    inv.dueDate != null &&
    inv.dueDate.getTime() + GRACE_DAYS * DAY_MS < now.getTime()
  );
}

/** Combine subscription + invoice state into a single block verdict. Suspended
 * subscriptions take precedence over past-due invoices. */
export function paymentBlockVerdict(input: {
  subscriptionStatus: string | null;
  hasPastGraceOverdue: boolean;
}): { blocked: boolean; reason: "suspended" | "past_due" | null } {
  if (isSuspended(input.subscriptionStatus)) return { blocked: true, reason: "suspended" };
  if (input.hasPastGraceOverdue) return { blocked: true, reason: "past_due" };
  return { blocked: false, reason: null };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run lib/billing/dunning.test.ts`
Expected: PASS (all cases).

- [ ] **Step 5: Full suite + typecheck**

Run: `npm run test && npx tsc --noEmit`
Expected: all green, no type errors.

- [ ] **Step 6: Commit**

```bash
git add lib/billing/dunning.ts lib/billing/dunning.test.ts
git commit -m "feat(billing): pure dunning decision helpers (1B)"
```

---

### Task 3: Enforcement helper + wire into ingest & the API guard

**Files:**
- Create: `lib/billing/enforcement.ts`
- Modify: `app/api/ingest/route.ts`
- Modify: `lib/api/guard.ts`

**Interfaces:**
- Consumes: `paymentBlockVerdict`, `isPastGrace` (Task 2); `invoice`/`tenantSettings` tables; `reportError` (`lib/observability`).
- Produces:
  ```ts
  export async function isOrgPaymentBlocked(organizationId: string): Promise<{
    blocked: boolean;
    reason: "suspended" | "past_due" | null;
    hasOverdueInvoice: boolean;
  }>;
  ```
  (Never throws — fails open to `{ blocked:false, reason:null, hasOverdueInvoice:false }` on error. `hasOverdueInvoice` is used by the tenant banner in Task 5.)

- [ ] **Step 1: Implement the helper**

Create `lib/billing/enforcement.ts`:

```ts
// lib/billing/enforcement.ts
// Org payment-block decision used by ingest + the /api/v1 guard + the tenant
// layout. One read of subscription status + overdue invoices; pure verdict from
// lib/billing/dunning. Fails OPEN on any error — never block a paying customer
// because of an infra fault.
import { and, eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { invoice as invoiceTable, tenantSettings } from "@/lib/db/schema";
import { paymentBlockVerdict, isPastGrace } from "@/lib/billing/dunning";
import { reportError } from "@/lib/observability";

export async function isOrgPaymentBlocked(organizationId: string): Promise<{
  blocked: boolean;
  reason: "suspended" | "past_due" | null;
  hasOverdueInvoice: boolean;
}> {
  try {
    const now = new Date();
    const [settings] = await db
      .select({ status: tenantSettings.subscriptionStatus })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, organizationId))
      .limit(1);

    const overdue = await db
      .select({ status: invoiceTable.status, dueDate: invoiceTable.dueDate })
      .from(invoiceTable)
      .where(
        and(eq(invoiceTable.organizationId, organizationId), eq(invoiceTable.status, "overdue")),
      );

    const hasOverdueInvoice = overdue.length > 0;
    const hasPastGraceOverdue = overdue.some((r) => isPastGrace(r, now));
    const verdict = paymentBlockVerdict({
      subscriptionStatus: settings?.status ?? null,
      hasPastGraceOverdue,
    });
    return { ...verdict, hasOverdueInvoice };
  } catch (err) {
    reportError(err, { path: "enforcement.isOrgPaymentBlocked", extra: { organizationId } });
    return { blocked: false, reason: null, hasOverdueInvoice: false };
  }
}
```

- [ ] **Step 2: Wire it into `/api/ingest`**

In `app/api/ingest/route.ts`, replace the imports `isSuspended` usage and the inline suspension block. Change the import line (currently `import { isSuspended } from "@/lib/billing/billing-status";`) to:

```ts
import { isOrgPaymentBlocked } from "@/lib/billing/enforcement";
```

Then replace the entire inline suspension `try { ... } catch { ... }` block (the one selecting `tenantSettings.subscriptionStatus` and calling `isSuspended`) with:

```ts
  // Block ingestion for orgs that are payment-blocked: suspended subscription
  // (403) or an unpaid invoice past the grace window (402). isOrgPaymentBlocked
  // fails open on a transient read error, so a DB blip can't lock out a customer.
  const block = await isOrgPaymentBlocked(device.organizationId);
  if (block.blocked) {
    return block.reason === "past_due"
      ? bad(402, "Account past due")
      : bad(403, "Subscription inactive");
  }
```

(If `tenantSettings` is now an unused import in this file, remove it from the `@/lib/db/schema` import list to keep `tsc`/lint clean. Leave `usageEvent`, `device`, `document` imports as they are.)

- [ ] **Step 3: Wire it into the `/api/v1` guard**

In `lib/api/guard.ts`, replace the `isSuspended` import with the helper and swap the inline block. Change `import { isSuspended } from "@/lib/billing/billing-status";` to:

```ts
import { isOrgPaymentBlocked } from "@/lib/billing/enforcement";
```

Replace the inline block (the `const [billing] = await db.select(...)` through the `if (isSuspended(...))` return) with:

```ts
  const block = await isOrgPaymentBlocked(auth.organizationId);
  if (block.blocked) {
    return block.reason === "past_due"
      ? { error: apiError("payment_past_due", "Account past due.", 402) }
      : { error: apiError("subscription_inactive", "Subscription inactive.", 403) };
  }
```

(Remove the now-unused `db`, `tenantSettings`, and `eq` imports from `guard.ts` if they are no longer referenced after this change — check the rest of the file; `apiError`, `authenticateApiKey`, `checkRateLimit` stay.)

- [ ] **Step 4: Typecheck + build + suite**

Run: `npx tsc --noEmit && npm run build && npm run test`
Expected: no type errors (watch for unused-import errors — remove any the edits orphaned); build OK; 259 green.

- [ ] **Step 5: Commit**

```bash
git add lib/billing/enforcement.ts app/api/ingest/route.ts lib/api/guard.ts
git commit -m "feat(billing): isOrgPaymentBlocked — enforce past-due (402) + suspended (403) at ingest + API (1B)"
```

---

### Task 4: Billing cron — generate → auto-send → sweep

**Files:**
- Modify: `lib/audit.ts` (add `AUDIT.invoiceOverdue`)
- Create: `lib/billing/billing-cron.ts`
- Create: `app/api/cron/billing/route.ts`
- Modify: `vercel.json`

**Interfaces:**
- Consumes: `runInvoiceGeneration(now: Date): Promise<InvoiceRun>` (`lib/billing-engine.ts`); `previousMonthMarker` (Task 2); `sendInvoiceToStripe(invoiceId)` (1A, `lib/billing/stripe-billing.ts`); `recordAudit`/`AUDIT` (`lib/audit.ts`); `invoice`/`tenantSettings` tables.
- Produces: `runBillingCron(now: Date): Promise<{ generated: number; autoSent: number; sweptOverdue: number }>`.

- [ ] **Step 1: Add the audit constant**

In `lib/audit.ts`, inside the `AUDIT` object after `invoiceSent: "invoice.sent",` add:

```ts
  invoiceOverdue: "invoice.overdue",
```

- [ ] **Step 2: Implement `runBillingCron`**

Create `lib/billing/billing-cron.ts`:

```ts
// lib/billing/billing-cron.ts
// The daily billing job (Phase 1B). Self-healing + idempotent: always processes
// the PREVIOUS calendar month, so a missed cron day catches up on the next run.
//   1. generate last month's draft invoices
//   2. auto-send drafts for tenants WITH a saved card (charge_automatically)
//   3. sweep sent invoices past their dueDate → overdue
import { and, eq, gte, lt, isNotNull } from "drizzle-orm";
import { db } from "@/lib/db";
import { invoice as invoiceTable, tenantSettings } from "@/lib/db/schema";
import { runInvoiceGeneration } from "@/lib/billing-engine";
import { previousMonthMarker } from "@/lib/billing/dunning";
import { sendInvoiceToStripe } from "@/lib/billing/stripe-billing";
import { recordAudit, AUDIT } from "@/lib/audit";

export async function runBillingCron(
  now: Date,
): Promise<{ generated: number; autoSent: number; sweptOverdue: number }> {
  const marker = previousMonthMarker(now);
  const periodStart = new Date(marker.getFullYear(), marker.getMonth(), 1);
  const periodEndExclusive = new Date(marker.getFullYear(), marker.getMonth() + 1, 1);

  // 1. Generate last month's drafts (idempotent — only touches draft rows).
  const run = await runInvoiceGeneration(marker);
  const generated = run.created + run.updated;

  // 2. Auto-send drafts for orgs that have a saved card (charge_automatically).
  const drafts = await db
    .select({ id: invoiceTable.id, cardLast4: tenantSettings.cardLast4 })
    .from(invoiceTable)
    .leftJoin(tenantSettings, eq(tenantSettings.organizationId, invoiceTable.organizationId))
    .where(
      and(
        eq(invoiceTable.status, "draft"),
        gte(invoiceTable.periodStart, periodStart),
        lt(invoiceTable.periodStart, periodEndExclusive),
      ),
    );
  let autoSent = 0;
  for (const d of drafts) {
    if (d.cardLast4 == null) continue; // no card → leave as draft for admin review
    const res = await sendInvoiceToStripe(d.id);
    if (res.ok) autoSent++;
  }

  // 3. Sweep sent invoices past due → overdue (+ audit each).
  const swept = await db
    .update(invoiceTable)
    .set({ status: "overdue" })
    .where(
      and(
        eq(invoiceTable.status, "sent"),
        isNotNull(invoiceTable.dueDate),
        lt(invoiceTable.dueDate, now),
      ),
    )
    .returning({
      id: invoiceTable.id,
      organizationId: invoiceTable.organizationId,
      dueDate: invoiceTable.dueDate,
    });
  for (const row of swept) {
    await recordAudit({
      organizationId: row.organizationId,
      actor: { type: "system" },
      action: AUDIT.invoiceOverdue,
      target: { type: "invoice", id: row.id },
      metadata: { dueDate: row.dueDate?.toISOString() ?? null },
    });
  }

  return { generated, autoSent, sweptOverdue: swept.length };
}
```

- [ ] **Step 3: Create the cron route**

Create `app/api/cron/billing/route.ts` (verbatim auth pattern from `credit-holds`):

```ts
// Daily billing job (Phase 1B): generate last month's invoices, auto-send card
// tenants, sweep overdue. Daily cadence (Vercel Hobby) + self-healing — always
// processes the previous calendar month, so a missed day catches up next run.
import { NextResponse } from "next/server";
import { getEnv } from "@/lib/env";
import { runBillingCron } from "@/lib/billing/billing-cron";

export const runtime = "nodejs";

export async function GET(req: Request) {
  const secret = getEnv().CRON_SECRET;
  if (!secret) return NextResponse.json({ error: "CRON_SECRET not configured" }, { status: 503 });
  if (req.headers.get("authorization") !== `Bearer ${secret}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }
  const result = await runBillingCron(new Date());
  return NextResponse.json({ ok: true, ...result });
}
```

- [ ] **Step 4: Register the cron schedule**

In `vercel.json`, add the billing entry to the `crons` array (after `credit-holds`):

```json
{
  "crons": [
    { "path": "/api/cron/health", "schedule": "0 9 * * *" },
    { "path": "/api/cron/webhooks", "schedule": "0 9 * * *" },
    { "path": "/api/cron/usage", "schedule": "0 9 * * *" },
    { "path": "/api/cron/credit-holds", "schedule": "0 9 * * *" },
    { "path": "/api/cron/billing", "schedule": "0 9 * * *" }
  ]
}
```

(Deploy note for the controller/user: if Vercel rejects a 5th cron on the current plan's count limit, fold these three steps into an existing daily cron route instead. This is a deploy-time concern, not a code change.)

- [ ] **Step 5: Typecheck + build + suite**

Run: `npx tsc --noEmit && npm run build && npm run test`
Expected: no type errors; build compiles the new route; 259 green.

- [ ] **Step 6: Commit**

```bash
git add lib/audit.ts lib/billing/billing-cron.ts app/api/cron/billing/route.ts vercel.json
git commit -m "feat(billing): daily billing cron — generate, auto-send card tenants, sweep overdue (1B)"
```

---

### Task 5: Tenant UX — overdue banner + blocked redirect

**Files:**
- Modify: `app/(tenant)/layout.tsx`

**Interfaces:**
- Consumes: `isOrgPaymentBlocked` (Task 3, returns `{ blocked, reason, hasOverdueInvoice }`).

- [ ] **Step 1: Use the helper for redirect + banner state**

In `app/(tenant)/layout.tsx`, replace the suspension import and the inline `subStatus` read + `isSuspended` redirect with the shared helper. Change the import `import { isSuspended } from "@/lib/billing/billing-status";` to:

```ts
import { isOrgPaymentBlocked } from "@/lib/billing/enforcement";
```

Replace the block from `// Billing enforcement...` through `const pastDue = subStatus === "past_due";` (the `try/catch` reading `subscriptionStatus`, the `isSuspended` redirect, and the `pastDue` line) with:

```ts
  // Billing enforcement. isOrgPaymentBlocked fails safe (no lock) on a read error.
  const payment = await isOrgPaymentBlocked(organizationId);
  const pathname = (await headers()).get("x-pathname") ?? "";
  if (payment.blocked && pathname !== "/tenant/billing") {
    redirect("/tenant/billing");
  }
  // Show the past-due banner whenever there is an unpaid overdue invoice (the
  // grace window before the hard block above kicks in).
  const pastDue = payment.hasOverdueInvoice;
```

(The existing `const pathname = ...` line below is now redundant — this edit moves it up; ensure there is only ONE `pathname` declaration in the file. The existing `{pastDue ? ( ... banner ... ) : null}` JSX stays as-is and now triggers on an overdue invoice.)

- [ ] **Step 2: Update the banner copy to match the invoice-based trigger**

The existing banner text references a failed payment. Update its wording in `app/(tenant)/layout.tsx` to fit an unpaid invoice:

```tsx
      {pastDue ? (
        <div className="mb-4 rounded-lg border border-amber-300 bg-amber-50 px-4 py-3 text-sm text-amber-900 dark:border-amber-900/50 dark:bg-amber-950/40 dark:text-amber-200">
          Your account has an unpaid invoice. Pay it on the{" "}
          <a href="/tenant/billing" className="font-medium underline">
            Billing
          </a>{" "}
          page to avoid interruption.
        </div>
      ) : null}
```

- [ ] **Step 3: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors (confirm only one `pathname` and one `subStatus`-free path remain — `tenantSettings`/`eq`/`db`/`isSuspended` imports are unused now and must be removed from this file if orphaned); build OK.

- [ ] **Step 4: Commit**

```bash
git add "app/(tenant)/layout.tsx"
git commit -m "feat(billing): tenant overdue banner + blocked redirect (1B)"
```

---

## Deferred acceptance steps (controller/user — prod-touching, after merge)

These are NOT subagent tasks (they mutate prod / need a running server):

1. **Apply the migration:** `npm run db:migrate` (adds the nullable `due_date` column to prod Neon — additive, safe).
2. **Cron smoke:** with `CRON_SECRET` set, `curl -H "Authorization: Bearer $CRON_SECRET" http://localhost:3001/api/cron/billing` → expect `{ ok:true, generated, autoSent, sweptOverdue }`.
3. **Enforcement smoke:** force an org into past-grace overdue (an `overdue` invoice with `dueDate` > 7 days ago) → confirm ingest returns **402 "Account past due"** and `/api/v1` returns the `payment_past_due` 402; a clean org still returns 201/200; a suspended org still returns 403.

---

## Self-Review

**Spec coverage:**
- `invoice.dueDate` column + populate on send (spec §A) → Task 1. ✅
- Pure helpers GRACE_DAYS/previousMonthMarker/isOverdue/isPastGrace/paymentBlockVerdict (spec §B) → Task 2. ✅
- `isOrgPaymentBlocked` + ingest/guard wiring, 402/403 (spec §C) → Task 3. ✅
- Billing cron: generate→auto-send card tenants→overdue sweep + `AUDIT.invoiceOverdue` + vercel.json (spec §D) → Task 4. ✅
- Tenant banner + blocked redirect (spec §E) → Task 5. ✅
- Fail-open enforcement, cron idempotency, no credit double-billing (spec error-handling) → Tasks 3 & 4. ✅
- Testing: pure unit tests (Task 2) + deferred IO smoke (matches spec §Testing). ✅

**Placeholder scan:** None — every code step shows complete code. ✅

**Type consistency:** `isOrgPaymentBlocked` return shape `{ blocked, reason, hasOverdueInvoice }` consistent across Tasks 3 & 5; `paymentBlockVerdict`/`isPastGrace` signatures (Task 2) match their callers in Task 3; `runBillingCron` return `{ generated, autoSent, sweptOverdue }` matches the cron route spread in Task 4; `previousMonthMarker` used identically in Tasks 2 & 4; `dueDate` column (Task 1) read in Tasks 3 & 4. ✅

**Note for implementers:** Tasks 3 & 5 each orphan imports (`isSuspended`, `tenantSettings`, `db`, `eq`) when the inline checks are removed — `tsc`'s no-unused-locals will flag them; remove the orphaned imports as part of the edit. The plan calls this out in each task.
