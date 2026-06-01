# Phase 1 — Billing MVP (Payment Collection) Design

_Last updated: 2026-06-01_

## Context

Ditto bills tenants **usage-based postpaid**: `lib/billing-engine.ts` meters
receipts and generates an idempotent monthly **draft** invoice per org
(`unitPriceCents` default $0.04/receipt). Today a platform admin manually
advances `draft → sent → paid` — **no money actually moves**. There is no
tenant-facing billing page and no Stripe integration.

This spec is the **first slice of Phase 1**: a complete, shippable vertical that
lets tenants save a card and get **auto-charged** for their monthly invoice via
Stripe, with status reconciled by webhooks. Dunning, suspension, `overdue/void`
states, and audit logs are explicitly **deferred to Spec 2**.

## Decisions (locked during brainstorming)

| Decision | Choice |
|---|---|
| Collection model | **Stripe Invoicing + saved card** — our engine stays the source of truth; Stripe collects off-session |
| Scope | **Payment-collection MVP** (this spec) |
| Card capture | **Embedded Stripe Elements** (`PaymentElement` + `SetupIntent`, `confirmSetup` in-app) |
| Charge trigger | **Scheduled cron** (Vercel Cron, monthly) |

## Goals

1. A tenant can add/update a card from their billing page (in-app Elements).
2. A monthly cron finalizes due draft invoices and charges the saved card off-session.
3. Webhooks reconcile Stripe outcomes back to `invoice.status`.
4. A tenant billing page shows their own invoices + saved card.
5. Everything builds and pure-unit-tests pass **without** Stripe keys (graceful degrade).

## Non-Goals (Spec 2)

`overdue`/`void` lifecycle states, dunning grace periods, account **suspension**
(blocking `/api/ingest` for unpaid orgs), retry schedules, payment-receipt
emails, and the **audit log** seed.

---

## The three flows

### A. Card capture (in-app, Stripe Elements)

1. Tenant opens `/tenant/billing`.
2. A server action `createSetupIntent(orgId)`:
   - `ensureStripeCustomer(orgId)` — find-or-create a Stripe Customer, store
     `tenantSettings.stripeCustomerId`.
   - Create a `SetupIntent` (`customer`, `usage: 'off_session'`); return its
     `client_secret`.
3. Client mounts `<PaymentElement>`, calls `stripe.confirmSetup({ redirect: 'if_required' })`.
4. On success, the saved payment method is set as the customer's
   `invoice_settings.default_payment_method`, and `cardBrand`/`cardLast4` are
   cached on `tenantSettings` (via the `setup_intent.succeeded` /
   `payment_method.attached` webhook, with a server-action fallback right after
   `confirmSetup`).

### B. Charge (Vercel Cron, monthly — `0 6 1 * *`)

`POST /api/cron/charge-invoices`, guarded by `Authorization: Bearer <CRON_SECRET>`:

1. `selectChargeableInvoices(invoices, now)` (pure) → draft invoices whose
   `periodEnd <= now` and `amountDueCents > 0` and not yet linked to Stripe.
2. For each, if the org has a default payment method:
   - `chargeInvoice(invoiceId)` — create a Stripe **Invoice**
     (`collection_method: 'charge_automatically'`, `customer`) with one invoice
     item = `amountDueCents`, then **finalize** it. Stripe attempts the
     off-session charge immediately.
   - Set our `invoice.status = 'sent'`, store `stripeInvoiceId`.
3. Orgs **without** a card → skipped and logged (Spec 2 dunning).

### C. Reconcile (webhook)

`POST /api/stripe/webhook` — verify signature with `STRIPE_WEBHOOK_SECRET`, then
`statusForStripeEvent(type)` (pure) maps:

| Stripe event | Action |
|---|---|
| `invoice.paid` | our invoice (by `stripeInvoiceId`) → `status = 'paid'` |
| `invoice.payment_failed` | leave `status = 'sent'` (Spec 2 → `overdue`) |
| `setup_intent.succeeded` / `payment_method.attached` | cache `cardBrand`/`cardLast4`, set default PM |

Idempotent: keyed on `stripeInvoiceId`; replaying an event is a no-op.

---

## Data model (additive — one migration, no enum change)

`tenantSettings` (PK = `organizationId`):
- `stripeCustomerId text` (nullable)
- `cardBrand text` (nullable)
- `cardLast4 text` (nullable)

`invoice`:
- `stripeInvoiceId text` (nullable) — links our row ↔ the Stripe Invoice

Status enum stays `draft | sent | paid`. `sent` is reinterpreted as
"finalized, charge in flight or unpaid". `overdue`/`void` arrive in Spec 2.

---

## File structure

| File | Responsibility | New? |
|---|---|---|
| `lib/stripe.ts` | Server Stripe client; `null` when `STRIPE_SECRET_KEY` unset | Create |
| `lib/billing/stripe-billing.ts` | `ensureStripeCustomer`, `createSetupIntent`, `chargeInvoice` + pure helpers `selectChargeableInvoices`, `statusForStripeEvent` | Create |
| `lib/billing/stripe-billing.test.ts` | Unit tests for the two pure helpers | Create |
| `app/api/stripe/webhook/route.ts` | Signature-verified event handler | Create |
| `app/api/cron/charge-invoices/route.ts` | Cron target, `CRON_SECRET`-guarded | Create |
| `app/(tenant)/tenant/billing/page.tsx` | Tenant billing page | Create |
| `app/(tenant)/tenant/billing/actions.ts` | `createSetupIntent` server action | Create |
| `components/billing/payment-method-form.tsx` | Client Elements form | Create |
| `lib/data.ts` | `getTenantBilling(orgId)` view-model | Modify |
| `lib/db/schema.ts` | Add the columns above | Modify |
| `lib/env.ts` | Add Stripe + cron env (all optional) | Modify |
| `vercel.json` | Cron schedule `0 6 1 * *` | Create |
| `.env.example` | Document the new vars | Modify |

---

## Error handling

- **Stripe not configured:** `lib/stripe.ts` exports `null`; the billing page
  renders "Billing isn't configured yet" instead of the Elements form; cron and
  webhook routes return 503. App still builds and boots.
- **Off-session charge fails** (declined / no card): Stripe fires
  `invoice.payment_failed`; we leave the invoice `sent`. No crash. Dunning is Spec 2.
- **Cron auth:** missing/wrong `CRON_SECRET` → 401.
- **Webhook:** bad signature → 400; unhandled event type → 200 (ignored).
- **Retries:** `chargeInvoice` skips invoices already linked to a `stripeInvoiceId`;
  webhook updates are idempotent by `stripeInvoiceId`.

## Testing

- **Pure unit (TDD, no Stripe needed, run green in CI):**
  - `selectChargeableInvoices(invoices, now)` — selects only due, positive-amount,
    unlinked drafts; excludes future periods, zero-amount, already-`sent`.
  - `statusForStripeEvent(type)` — maps the event table above; unknown → `null`.
- **Mocked integration:** webhook route with a hand-built event object (mock the
  Stripe client's `webhooks.constructEvent`); cron route rejects a bad secret.
- **Manual e2e (Stripe test mode):** `stripe listen --forward-to
  localhost:3000/api/stripe/webhook`; add card `4242 4242 4242 4242`; trigger the
  cron route by hand; use the off-session-decline test card to see
  `payment_failed`.

## Environment

| Var | Purpose | Required? |
|---|---|---|
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Client Elements | optional* |
| `STRIPE_SECRET_KEY` | Server Stripe client | optional* |
| `STRIPE_WEBHOOK_SECRET` | Webhook signature verification | optional* |
| `CRON_SECRET` | Authorize the cron route | optional* |

*Optional in the zod schema so the app builds without them; billing features are
inert until all are present. Test keys are already in `.env.local`
(`STRIPE_WEBHOOK_SECRET` filled from `stripe listen` at execution time).

## Sequencing

1. Schema + migration (additive columns).
2. `lib/stripe.ts` + env (guarded client).
3. Pure helpers + their tests (TDD).
4. `stripe-billing.ts` IO functions.
5. Webhook route.
6. Cron route + `vercel.json`.
7. Tenant billing page + Elements form + view-model.
8. Manual e2e in Stripe test mode.
