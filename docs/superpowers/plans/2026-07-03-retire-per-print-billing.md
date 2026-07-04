# Retire Per-Print Billing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the vestigial per-print (postpaid) billing subsystem so prepaid credits are the sole payment path, then drop the dead `document`/`usageEvent`/`invoice` tables and per-print columns.

**Architecture:** Top-down removal that keeps the build green at every task: rework the UI that references doomed code first, then delete the server actions those UIs called, then the Stripe functions/webhook branches those actions called, then the enforcement gate, then the synthetic revenue tiles, and finally the schema (once every reader is gone). Prepaid credits (`creditLedger`/`creditBalance`, the trigger reserve→settle flow, Stripe credit-pack Checkout) are untouched.

**Tech Stack:** Next.js 16 App Router, Drizzle over Neon HTTP, Better Auth (org plugin), Stripe (kept: Customers + one-time Checkout for credit packs), Vitest.

## Global Constraints

- **Prepaid credits are the sole payment path.** The trigger endpoint is already gated by `reserveCredit` → `402 insufficient_credits`; that is the only payment gate that remains.
- **KEEP (do not touch):** `lib/credits.ts`, `lib/credit-holds.ts`, `lib/credit-usage.ts`, `creditLedger` + `creditBalance` tables, the trigger endpoint + `app/api/device/commands/ack`, `lib/billing/credit-packs.ts`, `createCreditCheckout` + `startCreditCheckout` + `BuyCreditsSection`, the admin grant-credits form, `ensureStripeCustomer`, `tenantSettings.stripeCustomerId`, the `checkout.session.completed` webhook branch, `app/api/cron/credit-holds/route.ts`.
- **`.env.local` points at PRODUCTION Neon.** All verification is READ-ONLY (no INSERT/seed/write). The schema-drop migration (Task 9) is verified like migration 0028: confirm target tables/columns exist, apply at deploy, confirm dropped. Before the CASCADE drop, confirm the tables hold no real data (seed/test rows only).
- **Starter grant = 50 credits**, `kind='grant'`, idempotent by `idempotencyKey='starter-grant:<orgId>'`.
- **Audit enum entries** (`AUDIT.invoicePaid`, `subscriptionStatusChanged`, etc.) stay as historical constants; only their call sites are removed.
- **Each task ends green:** `npm run build` AND `npm test` pass before commit.
- **Deploy order (Task 9 onward):** ship code first (stops reading the tables), then `npm run db:migrate`.

---

### Task 1: Starter credit grant on org creation

Grant 50 credits when a tenant org is created, so new tenants can trigger without buying first. Additive and independent.

**Files:**
- Modify: `lib/actions/register.ts` (after `createOrganization` / tenantSettings seed, ~line 180)
- Modify: `lib/db/seed.ts` (after org creation, ~line 102-115)
- Create: `lib/credits.starter.test.ts`
- Reference: `lib/credits.ts` (`grantCredits({organizationId, credits, kind, idempotencyKey?, note?})` → `{applied: boolean}`)

**Interfaces:**
- Produces: `export const STARTER_CREDITS = 50;` in `lib/credits.ts`.

- [ ] **Step 1: Add the constant.** In `lib/credits.ts`, near the top exports, add:

```ts
/** Free credits granted once to a new tenant org so onboarding + first trigger work. */
export const STARTER_CREDITS = 50;
```

- [ ] **Step 2: Write the failing test.** Create `lib/credits.starter.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { STARTER_CREDITS } from "./credits";

describe("starter grant", () => {
  it("grants a fixed positive allotment", () => {
    expect(STARTER_CREDITS).toBe(50);
    expect(STARTER_CREDITS).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 3: Run it.** Run: `npm test -- credits.starter`. Expected: PASS (this pins the value; the grant wiring below is verified by build + prod smoke since it crosses Better-Auth/DB IO, which the codebase does not unit-test).

- [ ] **Step 4: Wire the grant into `registerCompany`.** In `lib/actions/register.ts`, import `grantCredits, STARTER_CREDITS` from `@/lib/credits`, and after the `dbTx.transaction(...)` block that seeds `tenantSettings` (so the org + settings exist), add:

```ts
    // Starter credits: prepaid is the only payment path, so a brand-new org
    // needs an allotment or its first trigger 402s. Idempotent by org id.
    await grantCredits({
      organizationId: orgId,
      credits: STARTER_CREDITS,
      kind: "grant",
      idempotencyKey: `starter-grant:${orgId}`,
      note: "starter grant",
    });
```

- [ ] **Step 5: Wire the grant into the seed.** In `lib/db/seed.ts`, after the org + owner membership are ensured, add the same `grantCredits({ organizationId: <orgId>, credits: STARTER_CREDITS, kind: "grant", idempotencyKey: \`starter-grant:${<orgId>}\`, note: "starter grant" })` call (idempotent, so re-seeding is safe). Import `grantCredits, STARTER_CREDITS` from `../credits` (match the file's existing import style).

- [ ] **Step 6: Build + test.** Run: `npm run build` then `npm test`. Expected: both PASS.

- [ ] **Step 7: Commit.**

```bash
git add lib/credits.ts lib/credits.starter.test.ts lib/actions/register.ts lib/db/seed.ts
git commit -m "feat(credits): grant 50 starter credits on org creation"
```

---

### Task 2: Repoint `/api/v1/usage` to credits

Rewrite the public usage endpoint off `document`/per-print onto credits + activations. Removes a `document` reader (needed before the Task 9 drop).

**Files:**
- Modify: `lib/data.ts` — `ApiUsageData` interface + `getApiUsage` (~1564-1616)
- Modify: `lib/api/serialize.ts` — `ApiUsage` + `serializeUsage`
- Modify: `lib/api/serialize.test.ts`
- Modify: `openapi.json` — `/usage` response schema (~lines 36-50)
- Reference: `getBalance(organizationId) → {available, held}` from `lib/credits.ts`; `creditLedgerTable`, `deviceCommand` already imported in `lib/data.ts`.

**Interfaces:**
- Produces: `ApiUsageData = { credits: {available:number; held:number}; creditsConsumedThisMonth:number; activationsThisMonth:number; period:{start:string; end:string} }`.

- [ ] **Step 1: Update the serializer test (TDD).** In `lib/api/serialize.test.ts`, replace the usage test's input/expected with the new shape:

```ts
    const u = {
      credits: { available: 120, held: 2 },
      creditsConsumedThisMonth: 48,
      activationsThisMonth: 48,
      period: { start: "2026-07-01T00:00:00.000Z", end: "2026-08-01T00:00:00.000Z" },
    };
    expect(serializeUsage(u)).toEqual({
      credits: { available: 120, held: 2 },
      credits_consumed_this_month: 48,
      activations_this_month: 48,
      period: { start: "2026-07-01T00:00:00.000Z", end: "2026-08-01T00:00:00.000Z" },
    });
```

- [ ] **Step 2: Run it (fails).** Run: `npm test -- serialize`. Expected: FAIL (old `serializeUsage` emits `documents_this_month`).

- [ ] **Step 3: Rewrite `serializeUsage`.** Replace `ApiUsage` + `serializeUsage` in `lib/api/serialize.ts`:

```ts
export interface ApiUsage {
  credits: { available: number; held: number };
  creditsConsumedThisMonth: number;
  activationsThisMonth: number;
  period: { start: string; end: string };
}

export function serializeUsage(u: ApiUsage) {
  return {
    credits: { available: u.credits.available, held: u.credits.held },
    credits_consumed_this_month: u.creditsConsumedThisMonth,
    activations_this_month: u.activationsThisMonth,
    period: { start: u.period.start, end: u.period.end },
  };
}
```

- [ ] **Step 4: Run it (passes).** Run: `npm test -- serialize`. Expected: PASS.

- [ ] **Step 5: Rewrite `getApiUsage`.** In `lib/data.ts`, add `import { getBalance } from "./credits";` (if not present) and replace the `ApiUsageData` interface + `getApiUsage` body (~1564-1616) with:

```ts
export interface ApiUsageData {
  credits: { available: number; held: number };
  creditsConsumedThisMonth: number;
  activationsThisMonth: number;
  period: { start: string; end: string };
}

/** Machine-keyed usage for /api/v1/usage — credit-denominated (UTC month). */
export async function getApiUsage(organizationId: string): Promise<ApiUsageData> {
  const now = new Date();
  const monthStart = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), 1));
  const monthEnd = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth() + 1, 1));

  const [credits, [consumedRow], [actRow]] = await Promise.all([
    getBalance(organizationId),
    db
      .select({ c: sql<number>`coalesce(sum(${creditLedgerTable.credits}), 0)::int` })
      .from(creditLedgerTable)
      .where(and(
        eq(creditLedgerTable.organizationId, organizationId),
        eq(creditLedgerTable.kind, "settle"),
        gte(creditLedgerTable.createdAt, monthStart),
      )),
    db
      .select({ c: count() })
      .from(deviceCommand)
      .where(and(
        eq(deviceCommand.organizationId, organizationId),
        eq(deviceCommand.type, "trigger"),
        eq(deviceCommand.status, "acked"),
        gte(deviceCommand.createdAt, monthStart),
      )),
  ]);

  return {
    credits,
    creditsConsumedThisMonth: Number(consumedRow?.c ?? 0),
    activationsThisMonth: Number(actRow?.c ?? 0),
    period: { start: monthStart.toISOString(), end: monthEnd.toISOString() },
  };
}
```

- [ ] **Step 6: Update the OpenAPI `/usage` schema.** In `openapi.json`, replace the `properties` of the `/usage` 200 response (the `unit_price_cents`/`documents_this_month`/`current_period`/`daily`/`monthly` block) with:

```json
          "credits": { "type": "object", "properties": { "available": { "type": "integer" }, "held": { "type": "integer" } } },
          "credits_consumed_this_month": { "type": "integer" },
          "activations_this_month": { "type": "integer" },
          "period": { "type": "object", "properties": { "start": { "type": "string" }, "end": { "type": "string" } } }
```

- [ ] **Step 7: Build + test.** Run: `npm run build` then `npm test`. Expected: both PASS. If `openapi.test.ts` asserts the old fields, update its expectations to the new keys.

- [ ] **Step 8: Commit.**

```bash
git add lib/data.ts lib/api/serialize.ts lib/api/serialize.test.ts openapi.json lib/api/openapi.test.ts
git commit -m "feat(api): repoint /v1/usage from documents to credits"
```

---

### Task 3: Rework the tenant billing page (strip invoices + subscription)

Remove the invoices table, subscription header, and "Activate billing" card from the tenant billing page; keep credits balance, purchase, and usage. Removes the UI's references to `activateBilling` + invoice data.

**Files:**
- Modify: `app/(tenant)/tenant/billing/page.tsx`
- Modify: `app/(tenant)/tenant/billing/actions.ts` (remove `activateBilling` server action; keep `startCreditCheckout`)
- Delete: `components/billing/payment-method-form.tsx`
- Modify: `lib/data.ts` — `getTenantBilling` (drop the invoices/subscription portions; keep balance + credit-usage)

**Interfaces:**
- Consumes: `getBalance`, `getCreditUsageByDevice` (existing), `BuyCreditsSection`.
- Produces: `getTenantBilling` returns only credit-relevant fields (no `invoices`, no `subscriptionStatus`/`hasSubscription`).

- [ ] **Step 1: Trim `getTenantBilling`.** In `lib/data.ts`, remove the invoice query + `subscriptionStatus`/`hasSubscription`/`cardBrand`/`cardLast4` fields from `getTenantBilling`'s return; keep the credit balance + credit-usage-by-device parts. (Note the exact returned shape — the page in Step 2 must match it.)

- [ ] **Step 2: Rework the page.** In `app/(tenant)/tenant/billing/page.tsx`, remove: the `PaymentMethodForm` import + render, the subscription-status header block, and the Invoices `<table>` (the `billing.invoices` map). Keep the balance card, `<BuyCreditsSection>`, and the credit-usage-by-device table. Ensure no reference to removed `billing.*` fields remains.

- [ ] **Step 3: Remove the `activateBilling` server action.** In `app/(tenant)/tenant/billing/actions.ts`, delete the `activateBilling` export (and its import of `activateBilling` from `stripe-billing`). Keep `startCreditCheckout`.

- [ ] **Step 4: Delete the component.** `git rm components/billing/payment-method-form.tsx`.

- [ ] **Step 5: Build.** Run: `npm run build`. Expected: PASS. Fix any dangling `PaymentMethodForm`/`billing.invoices`/`billing.subscriptionStatus` references the compiler flags.

- [ ] **Step 6: Verify (read-only).** Run the tenant billing route in a build and confirm no type errors; the live smoke happens at deploy. Run `npm test`. Expected: PASS.

- [ ] **Step 7: Commit.**

```bash
git add "app/(tenant)/tenant/billing" lib/data.ts
git rm components/billing/payment-method-form.tsx
git commit -m "refactor(billing): strip invoices + subscription from tenant billing page"
```

---

### Task 4: Rework the admin billing page to a credits view

Replace the invoice-based admin billing page (earnings/MRR/outstanding + revenue chart + invoices table) with a credits view. Removes the UI's references to `getBillingOverview` + the invoice action buttons.

**Files:**
- Create: `lib/credits-overview.ts` (pure rollup helper) + `lib/credits-overview.test.ts`
- Modify: `lib/data.ts` — add `getCreditsOverview(): Promise<CreditsOverview>` (IO; feeds the pure helper)
- Modify: `app/(admin)/admin/billing/page.tsx` (rewrite to the credits view)

**Interfaces:**
- Produces: `CreditsOverview = { totals: {granted:number; purchased:number; consumed:number; outstanding:number}; perTenant: {orgId:string; name:string; balance:number; consumedThisMonth:number; lifetimePurchased:number}[] }`.

- [ ] **Step 1: Write the pure rollup + test.** Create `lib/credits-overview.ts` with a pure `rollupCredits(ledgerRows, balances, now)` that folds ledger rows (`kind` ∈ grant/purchase/settle) into the `CreditsOverview` totals + per-tenant rows, and `lib/credits-overview.test.ts` asserting: granted sums `grant` credits, purchased sums `purchase`, consumed sums `settle`, `outstanding` sums balances, `consumedThisMonth` filters `settle` rows to `>= startOfMonth(now)`. (Write concrete rows + expected object — mirror the style of `lib/credit-usage.test.ts`.)

- [ ] **Step 2: Run it (fails → passes).** Run: `npm test -- credits-overview`. Iterate until PASS.

- [ ] **Step 3: Add `getCreditsOverview` (IO).** In `lib/data.ts`, add `getCreditsOverview()` that selects all orgs, all `creditLedger` rows (org, kind, credits, createdAt), and `creditBalance` per org, then calls `rollupCredits(...)`. Sort `perTenant` by `balance` desc.

- [ ] **Step 4: Rewrite the admin billing page.** In `app/(admin)/admin/billing/page.tsx`, replace the invoice KPIs/`RevenueLineChart`/invoices table with: KPI cards (credits sold = purchased, credits consumed, outstanding liability), and a per-tenant table (name, balance, consumed this month, lifetime purchased). Remove imports of `GenerateInvoicesButton`, `InvoiceRowActions`, `getBillingOverview`, `RevenueLineChart`.

- [ ] **Step 5: Build + test.** Run: `npm run build` then `npm test`. Expected: both PASS.

- [ ] **Step 6: Commit.**

```bash
git add lib/credits-overview.ts lib/credits-overview.test.ts lib/data.ts "app/(admin)/admin/billing/page.tsx"
git commit -m "feat(billing): rework admin billing into a credits overview"
```

---

### Task 5: Delete the invoice generation engine, crons, and admin invoice actions

With both billing UIs reworked, delete the now-unreferenced per-print generation/collection code and its tests.

**Files (delete):** `lib/billing-engine.ts`, `lib/billing/billing-cron.ts`, `lib/billing/invoice-collect.ts`, `lib/billing/usage-metering.ts`, `lib/billing/usage-status.ts`, `app/api/cron/billing/route.ts`, `app/api/cron/usage/route.ts`, `components/generate-invoices-button.tsx`, `components/invoice-row-actions.tsx`, and tests `lib/billing/invoice-collect.test.ts`, `lib/billing/usage-metering.test.ts`.
**Files (modify):** `lib/actions/billing.ts` (remove `generateInvoices`, `advanceInvoice`, `setInvoiceStatus`, `voidInvoice`; keep the file only if something credit-related remains, else delete it), `lib/data.ts` (remove `getBillingOverview`), `vercel.json`/cron config if it lists the deleted cron paths.

- [ ] **Step 1: Delete the files.** `git rm` each file in the delete list.

- [ ] **Step 2: Remove invoice server actions.** Edit `lib/actions/billing.ts` to drop the four invoice actions + their imports (`runInvoiceGeneration`, `sendInvoiceToStripe`, etc.). If the file is now empty, `git rm` it.

- [ ] **Step 3: Remove `getBillingOverview`.** Delete the `getBillingOverview` function + its `BillingOverview` type from `lib/data.ts` (Task 4 removed its only caller).

- [ ] **Step 4: Remove deleted cron paths from config.** If `vercel.json` (or `vercel.ts`) declares `/api/cron/billing` or `/api/cron/usage` schedules, remove those entries. Keep `/api/cron/credit-holds`.

- [ ] **Step 5: Build + test + grep.** Run: `npm run build` then `npm test`. Then `grep -rn "billing-engine\|runInvoiceGeneration\|generateInvoices\|getBillingOverview\|billing-cron\|usage-metering" app/ lib/ components/` → expect no output (except historical audit-label strings). Expected: build+test PASS.

- [ ] **Step 6: Commit.**

```bash
git add -A
git commit -m "refactor(billing): remove per-print invoice engine, crons, and admin invoice actions"
```

---

### Task 6: Trim the Stripe webhook + stripe-billing to credits-only

Remove the invoice/subscription/card handling; keep customer creation + credit-pack checkout + the credit-purchase webhook branch.

**Files:**
- Modify: `app/api/stripe/webhook/route.ts`
- Modify: `lib/billing/stripe-billing.ts` (remove `activateBilling`, `reportDocumentUsage`, `sendInvoiceToStripe`; keep `ensureStripeCustomer`, `createCreditCheckout`)
- Delete: `lib/billing/invoice-sync.ts`, `lib/billing/invoice-emails.ts`, `lib/billing/billing-status.ts`, and tests `lib/billing/invoice-sync.test.ts`, `lib/billing/invoice-emails.test.ts`; trim `lib/billing/stripe-billing.test.ts` to the kept functions.

- [ ] **Step 1: Replace the webhook with the credits-only version.** Overwrite `app/api/stripe/webhook/route.ts` with (keeps only the guards + `checkout.session.completed`):

```ts
// POST /api/stripe/webhook — reconcile Stripe events into our DB.
// Verified by STRIPE_WEBHOOK_SECRET. Credit-pack purchase is the only event we act on.
import { NextResponse } from "next/server";
import type Stripe from "stripe";
import { stripe } from "@/lib/stripe";
import { getEnv } from "@/lib/env";
import { grantCredits } from "@/lib/credits";
import { recordAudit, AUDIT } from "@/lib/audit";

export const runtime = "nodejs";

export async function POST(req: Request) {
  if (!stripe) return NextResponse.json({ error: "billing not configured" }, { status: 503 });
  const secret = getEnv().STRIPE_WEBHOOK_SECRET;
  if (!secret) return NextResponse.json({ error: "no webhook secret" }, { status: 503 });

  const sig = req.headers.get("stripe-signature") ?? "";
  const body = await req.text();

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, secret);
  } catch {
    return NextResponse.json({ error: "bad signature" }, { status: 400 });
  }

  if (event.type === "checkout.session.completed") {
    const session = event.data.object as Stripe.Checkout.Session;
    if (
      session.mode === "payment" &&
      session.payment_status === "paid" &&
      session.metadata?.organizationId
    ) {
      const credits = Number(session.metadata.credits);
      if (Number.isInteger(credits) && credits > 0) {
        const res = await grantCredits({
          organizationId: session.metadata.organizationId,
          credits,
          kind: "purchase",
          idempotencyKey: session.id,
          note: `stripe pack ${session.metadata.packId ?? ""}`,
        });
        if (res.applied) {
          await recordAudit({
            organizationId: session.metadata.organizationId,
            actor: { type: "stripe" },
            action: AUDIT.creditsPurchased,
            metadata: { credits, sessionId: session.id },
          });
        }
      }
    }
  }

  return NextResponse.json({ received: true });
}
```

- [ ] **Step 2: Trim `stripe-billing.ts`.** Remove `activateBilling`, `reportDocumentUsage`, `sendInvoiceToStripe` and any imports only they used (`STRIPE_PRICE_ID`, meter helpers). Keep `ensureStripeCustomer`, `createCreditCheckout`.

- [ ] **Step 3: Delete the invoice/email/status modules + tests.** `git rm lib/billing/invoice-sync.ts lib/billing/invoice-emails.ts lib/billing/billing-status.ts lib/billing/invoice-sync.test.ts lib/billing/invoice-emails.test.ts`. In `lib/billing/stripe-billing.test.ts`, delete cases for the removed functions (keep customer/checkout cases); if none remain, `git rm` it.

- [ ] **Step 4: Build + test + grep.** Run: `npm run build` then `npm test`. Then `grep -rn "activateBilling\|reportDocumentUsage\|sendInvoiceToStripe\|invoice-sync\|invoice-emails\|billing-status" app/ lib/ components/` → expect no output. Expected: PASS.

- [ ] **Step 5: Commit.**

```bash
git add -A
git commit -m "refactor(stripe): trim webhook + stripe-billing to credit-pack purchases only"
```

---

### Task 7: Delete the payment-enforcement gate

Remove `isOrgPaymentBlocked` and everything that consumed it. Credits remain the only gate.

**Files:**
- Delete: `lib/billing/enforcement.ts`, `lib/billing/dunning.ts`, `lib/billing/dunning.test.ts`
- Modify: `lib/api/guard.ts`, `app/(tenant)/layout.tsx`

- [ ] **Step 1: Remove the gate from `guard.ts`.** Replace the enforcement block so `guardApiRequest` ends after the rate-limit check:

```ts
import { NextResponse } from "next/server";
import { authenticateApiKey, type ApiKeyAuth } from "@/lib/api-auth";
import { checkRateLimit } from "@/lib/rate-limit";
import { apiError } from "@/lib/api/respond";

export async function guardApiRequest(
  req: Request,
): Promise<{ auth: ApiKeyAuth } | { error: NextResponse }> {
  const auth = await authenticateApiKey(req);
  if (!auth) return { error: apiError("unauthorized", "Missing or invalid API key.", 401) };

  const rl = await checkRateLimit(auth.keyHash, { limit: 120, windowMs: 60_000 });
  if (!rl.allowed) {
    const res = apiError("rate_limited", "Too many requests.", 429);
    res.headers.set("Retry-After", String(Math.ceil(rl.retryAfterMs / 1000)));
    return { error: res };
  }

  return { auth };
}
```

- [ ] **Step 2: Remove the tenant-layout redirect/banner.** In `app/(tenant)/layout.tsx`, delete the `isOrgPaymentBlocked` import + call, the redirect-to-`/tenant/billing` branch, and the overdue banner (`hasOverdueInvoice`). Keep the rest of the layout intact.

- [ ] **Step 3: Delete the modules.** `git rm lib/billing/enforcement.ts lib/billing/dunning.ts lib/billing/dunning.test.ts`.

- [ ] **Step 4: Build + test + grep.** Run: `npm run build` then `npm test`. Then `grep -rn "isOrgPaymentBlocked\|enforcement\|paymentBlockVerdict\|isPastGrace" app/ lib/` → expect no output. Expected: PASS.

- [ ] **Step 5: Commit.**

```bash
git add -A
git commit -m "refactor(billing): delete payment-enforcement gate (credits self-enforce)"
```

---

### Task 8: Remove synthetic revenue tiles + per-print price reads

`revenue` (`activations × perPrintPriceCents`) was always a synthetic estimate; with per-print gone it has no basis. Remove it end-to-end. Atomic (a partial removal breaks the build). TDD via `analytics.test.ts`.

**Files:** `lib/types.ts` (`TimePoint.revenue`, `Invoice.*` already dropped in Task 5/9 — leave `Invoice` type until Task 9), `lib/analytics.ts` + `lib/analytics.test.ts`, `components/charts.tsx` (`RevenueLineChart`), `lib/data.ts` (`revenueThisMonth`, `perPrintPrice` reads in `summarize`/`buildTenant`/`getStoreAnalytics`/`getStoresAnalytics`), and UI: `app/(admin)/admin/page.tsx`, `app/(admin)/admin/customers/page.tsx`, `app/(admin)/admin/customers/[tenantId]/page.tsx`, `app/(tenant)/tenant/analytics/page.tsx`, `app/(tenant)/tenant/reports/page.tsx`, `app/(tenant)/tenant/stores/[storeId]/page.tsx`.

- [ ] **Step 1: Update `analytics.test.ts` (TDD).** Remove `revenue` from the `bucketsToSeries` expected `TimePoint` objects and `revenueThisMonth` from `toComparisonRows`/`StoreAnalytics` expectations. Run `npm test -- analytics` → FAIL.

- [ ] **Step 2: Remove `revenue` from `lib/analytics.ts`.** In `bucketsToSeries`, return `{ label, activations }` (drop `revenue`); drop the `price` param if now unused (and its callers pass none). Remove `revenueThisMonth` from `StoreAnalytics`/`StoreComparisonRow`/`toComparisonRows`. Update `lib/types.ts` `TimePoint` to `{ label: string; activations: number }`.

- [ ] **Step 3: Run analytics test → PASS.** Run: `npm test -- analytics`.

- [ ] **Step 4: Remove revenue in `lib/data.ts`.** Drop `revenueThisMonth` and every `tenant.perPrintPrice`/`perPrintPriceCents`-derived revenue computation from `summarize`, `buildTenant`, `getStoreAnalytics`, `getStoresAnalytics`, `getTenantDashboard`, and the series builders (`dailySeries`/`monthlySeries`/`sumSeries` no longer compute revenue). Remove `perPrintPrice` from the `Tenant`/`TenantSummary` view models in `lib/types.ts`.

- [ ] **Step 5: Remove `RevenueLineChart` + revenue UI.** Delete `RevenueLineChart` from `components/charts.tsx`; remove its imports/renders and the "Revenue" KPI cards / table columns from the six pages listed above (admin overview "Revenue over time" + "Revenue" column; admin customer "Revenue (mo.)"; tenant analytics/reports revenue columns + CSV headers; store detail revenue tile).

- [ ] **Step 6: Build + test + grep.** Run: `npm run build` then `npm test`. Then `grep -rn "revenueThisMonth\|RevenueLineChart\|perPrintPrice\|TimePoint.*revenue\|\.revenue\b" app/ components/ lib/ | grep -v invoice` → expect no output. Expected: PASS.

- [ ] **Step 7: Commit.**

```bash
git add -A
git commit -m "refactor(metrics): remove synthetic per-print revenue tiles"
```

---

### Task 9: Drop the vestigial schema + final cleanup

Every reader of `document`/`usageEvent`/`invoice` and the per-print columns is now gone. Drop them.

**Files:**
- Modify: `lib/db/schema.ts` (remove `document`, `usageEvent`, `invoice` tables + their exports; remove `tenantSettings` columns `perPrintPriceCents`, `stripeSubscriptionId`, `subscriptionStatus`, `cardBrand`, `cardLast4`; KEEP `stripeCustomerId`)
- Modify: `lib/db/relations.ts` (remove `documentRelations` + inbound refs)
- Modify: `lib/db/seed.ts` (remove the document-seeding block + `documentToken` usage)
- Modify: `scripts/verify.ts` (remove `count(document)`), `lib/api-scopes.ts` (remove the orphaned `documents:read` scope), `lib/env.ts` + `.env.example` (remove `STRIPE_PRICE_ID`, `STRIPE_METER_EVENT_NAME`)
- Modify: `lib/types.ts` (remove the now-unused `Invoice`/`InvoiceStatus`/`InvoiceLifecycle` types)
- Generate: one Drizzle migration under `drizzle/`

- [ ] **Step 1: Pre-drop read-only safety check.** Write a scratchpad script (pattern from prior deploys: import `./lib/db/load-env.ts`, `neon`, `sql.query`) that counts rows in `invoice`, `document`, `usage_event` on prod. Run it. Record the counts. Proceed only if they are seed/test data (no real customer invoices). Delete the script.

- [ ] **Step 2: Remove the schema objects.** In `lib/db/schema.ts` delete the three `pgTable` definitions + remove them from the `schema` export map; delete the five `tenantSettings` columns. In `lib/db/relations.ts` remove `documentRelations` and any `document`/`invoice`/`usageEvent` references in other relations. Remove the now-dead imports.

- [ ] **Step 3: Clean the remaining readers.** Remove the document-seeding block in `lib/db/seed.ts`; remove `count(document)` from `scripts/verify.ts`; remove `documents:read` from `lib/api-scopes.ts` (+ any test asserting the scope list); remove `STRIPE_PRICE_ID`/`STRIPE_METER_EVENT_NAME` from `lib/env.ts` + `.env.example`; remove the `Invoice*` types from `lib/types.ts`.

- [ ] **Step 4: Generate the migration.** Run: `npm run db:generate`. Inspect the emitted SQL — it must be exactly the 3 `DROP TABLE ... CASCADE` + 5 `ALTER TABLE tenant_settings DROP COLUMN`. Strip any spurious FK/index churn per [[drizzle-snapshot-drift]] (hand-edit the `.sql` to only your intended change; keep the drizzle snapshot meta consistent).

- [ ] **Step 5: Build + test.** Run: `npm run build` then `npm test`. Expected: PASS (no references to the dropped objects remain).

- [ ] **Step 6: Commit (do NOT migrate prod here — that is the deploy step).**

```bash
git add -A
git commit -m "refactor(db): drop document/usageEvent/invoice tables + per-print columns"
```

---

## Rollout (after all tasks reviewed + merged to main)

1. `vercel --prod --yes` (ships code that no longer reads the dropped tables/columns).
2. `npm run db:migrate` (applies the drop migration to prod Neon).
3. Read-only smoke: new signup receives 50 credits and can trigger; `GET /api/v1/usage` returns the credit shape; tenant + admin billing pages render; `information_schema` confirms the tables/columns are gone.

## Self-review notes

- **Spec coverage:** Decision 1 (delete gate) → Task 7; Decision 2 (starter grant) → Task 1; Decision 3 (/v1/usage credits) → Task 2; Decision 4 (admin credits view) → Task 4; Decision 5 (remove revenue) → Task 8; remove engine/cron/stripe → Tasks 5–6; drop schema → Task 9; keep-list honored throughout (credits path never touched).
- **Ordering keeps build green:** UI reworks (3,4) before action deletes (5) before stripe-fn deletes (6); enforcement (7) independent; revenue (8) atomic; schema drop (9) last, after all readers gone.
- **Placeholder scan:** surgical tasks carry full code; deletion tasks carry exact paths + grep gates. Tasks 3/4/8's per-page edits are enumerated by file with the exact elements to remove (no vague "clean up").
- **Type consistency:** `ApiUsageData`/`ApiUsage` shape defined once (Task 2) and matched in serialize; `CreditsOverview` defined in Task 4; `TimePoint` becomes `{label, activations}` in Task 8 and every consumer is enumerated.
