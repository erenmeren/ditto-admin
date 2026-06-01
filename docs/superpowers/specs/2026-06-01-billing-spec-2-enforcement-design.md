# Billing Spec 2 — Enforcement, Lifecycle & Audit Log Design

_Last updated: 2026-06-01_

## Context

Billing Spec 1 (shipped) collects money: tenants subscribe via Stripe Checkout
(metered subscription), usage is metered per receipt, and Stripe-generated
invoices are mirrored into our `invoice` table via webhooks. Stripe also runs
**dunning** (Smart Retries) on failed payments, walking a subscription
`active → past_due → canceled/unpaid`, which we already mirror into
`tenantSettings.subscriptionStatus`.

This spec makes billing **enforceable**: react to Stripe's terminal-unpaid
signal by **suspending** the org, add the missing invoice lifecycle states, and
introduce an **audit log** of notable account/billing/operational events.

## Decisions (locked during brainstorming)

| Decision | Choice |
|---|---|
| Scope | Enforcement **+** invoice lifecycle **+** audit log (one spec) |
| Suspend trigger | **Terminal unpaid only** — `subscriptionStatus ∈ {canceled, unpaid, incomplete_expired}`. `past_due` is NOT suspended (Stripe still retrying) |
| Suspend severity | **Block ingest + lock dashboard** — suspended orgs can only reach `/tenant/billing` |
| Coverage | **Only lapsed subscribers** — `null`/`active`/`past_due`/`trialing` are allowed; never-subscribed orgs are unaffected |
| Audit events | **Everything notable** — billing/subscription, org suspend/reactivate, device CRUD, store create, branding, signup. (Member CRUD deferred: no member-management flow exists yet) |
| Audit views | **Platform admin (any org) + tenant (own org)** |

## Goals

1. A suspended org (terminal-unpaid subscription) cannot ingest receipts and is
   locked to the billing page until it reactivates.
2. A `past_due` org keeps working but sees a warning banner.
3. Invoices reflect `overdue` and `void` states.
4. Notable events are recorded to an audit log, viewable by platform admin
   (per org) and by tenants (their own org).
5. Pure logic (`isSuspended`, status mapping) is unit-tested without Stripe/DB.

## Non-Goals

- Member management + member-change auditing (no member UI exists; deferred).
- In-app dunning timers / custom grace windows (Stripe owns retries).
- Email notifications on suspension (future).

---

## 1. Enforcement (suspension)

### Derived state (no new column)

```ts
// lib/billing/billing-status.ts (pure)
const SUSPENDED = new Set(["canceled", "unpaid", "incomplete_expired"]);
export function isSuspended(subscriptionStatus: string | null): boolean {
  return subscriptionStatus != null && SUSPENDED.has(subscriptionStatus);
}
```

`tenantSettings.subscriptionStatus` (webhook-maintained) is the single source of
truth. Suspension is derived at read time — nothing extra to keep in sync.

### Ingest block

In `/api/ingest`, after device authentication, read the org's
`subscriptionStatus` and reject if suspended:

```ts
const [settings] = await db
  .select({ status: tenantSettings.subscriptionStatus })
  .from(tenantSettings)
  .where(eq(tenantSettings.organizationId, device.organizationId))
  .limit(1);
if (isSuspended(settings?.status ?? null)) return bad(403, "Subscription inactive");
```

This read also serves the existing metering lookup (consolidate the two reads).

### Dashboard lock

- `middleware.ts` (already matches `/tenant`) injects the request path as an
  `x-pathname` header via `NextResponse.next({ request: { headers } })`. No DB —
  stays an optimistic gate.
- The tenant layout (`app/(tenant)/layout.tsx`) reads `headers().get("x-pathname")`
  and the org's `subscriptionStatus`. If `isSuspended(...)` and the path is not
  `/tenant/billing`, `redirect("/tenant/billing")`.
- `/tenant/billing` renders a "subscription canceled — reactivate to continue"
  state (reuses the existing Activate-billing Checkout flow; `ensureStripeCustomer`
  reuses the same customer).
- A `past_due` (not suspended) org sees a warning **banner** in the layout but is
  not locked.

### Reactivation

Paying / re-subscribing → Stripe `customer.subscription.updated` (status
`active`) → webhook updates `subscriptionStatus` → `isSuspended` returns false →
access restored. Stripe `canceled` is terminal, so recovery = create a new
subscription through the existing Checkout flow.

---

## 2. Invoice lifecycle (overdue / void)

- Widen the `invoice.status` TS enum to `draft | sent | paid | overdue | void`.
  **No migration** — the DB column is plain `text`; the enum is type-level only.
- Extend the pure mapper:

```ts
export function statusForStripeInvoice(stripeStatus: string): InvoiceStatus {
  switch (stripeStatus) {
    case "draft": return "draft";
    case "paid": return "paid";
    case "void": return "void";
    case "uncollectible": return "overdue";
    default: return "sent"; // open
  }
}
```

- Webhook: on `invoice.payment_failed`, upsert then set status `overdue`
  (the Stripe invoice is still `open`, so the event drives the override). On
  `invoice.voided`/status `void` → `void` via the mapper.

---

## 3. Audit log

### Table (`lib/db/schema.ts`)

```ts
export const auditLog = pgTable("audit_log", {
  id: text("id").primaryKey(),
  organizationId: text("organization_id").notNull()
    .references(() => organization.id, { onDelete: "cascade" }),
  actorType: text("actor_type", { enum: ["user", "system", "stripe"] }).notNull(),
  actorId: text("actor_id"),
  actorLabel: text("actor_label"),
  action: text("action").notNull(),
  targetType: text("target_type"),
  targetId: text("target_id"),
  metadata: jsonb("metadata"),
  createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
}, (t) => [index("audit_log_org_created_idx").on(t.organizationId, t.createdAt)]);
```

### Writer (`lib/audit.ts`)

```ts
type Actor =
  | { type: "user"; id: string; label: string }
  | { type: "system" }
  | { type: "stripe" };

export async function recordAudit(input: {
  organizationId: string;
  actor: Actor;
  action: string;
  target?: { type: string; id: string };
  metadata?: Record<string, unknown>;
}): Promise<void> { /* insert one row; never throws into the caller's path */ }
```

`recordAudit` is best-effort: it catches its own errors and logs, so auditing a
device delete never fails the delete.

### Actions (string constants)

`org.suspended`, `org.reactivated`, `subscription.status_changed`,
`invoice.paid`, `invoice.payment_failed`, `invoice.void`, `org.created`,
`device.provisioned`, `device.renamed`, `device.reassigned`, `device.deleted`,
`device.paused`, `device.resumed`, `store.created`, `device.claimed`,
`branding.updated`, `billing.activated`, `customer.created`.

### Write-sites

- **Webhook** (`actor: stripe`/`system`): on `customer.subscription.*`, read the
  previous `subscriptionStatus`, compare to the new one — record
  `subscription.status_changed` (metadata `{from,to}`); if the transition crosses
  into the suspended set record `org.suspended`, if out of it record
  `org.reactivated`. On invoice events record `invoice.paid|payment_failed|void`.
- **Server actions** (`actor: user`, from `requireTenant()`/`requirePlatformAdmin()`
  session): device actions (`lib/actions/devices.ts`), store create + device claim
  (`lib/actions/stores.ts`, `stores/[storeId]/actions.ts`), branding
  (`branding/actions.ts`), billing activate (`billing/actions.ts`), admin
  customer-create (`lib/actions/customers.ts`), signup (`lib/actions/register.ts`
  → `org.created`).

### Views

- **Platform admin:** an "Activity" section on the customer detail page
  (`app/(admin)/admin/customers/[tenantId]/page.tsx`) listing that org's audit
  rows (newest first), via a `getOrgAuditLog(orgId, limit)` data-layer fn.
- **Tenant:** a new `/tenant/activity` page (+ `TENANT_NAV` link) showing the
  org's own rows. Locked when suspended (only billing reachable).

---

## 4. Data model summary

- **New:** `auditLog` table (one additive migration).
- **Type-only:** `invoice.status` enum widened (no migration).
- No other column changes; `jsonb` import added to schema if not present.

## 5. Error handling

- `recordAudit` swallows its own errors (best-effort; never breaks the audited action).
- Ingest suspension check fails open only on a DB read error? No — on read error
  it should fail safe (allow), to avoid blocking paying customers on a transient
  DB blip; log the error. (Ingest already tolerates metering failures.)
- Layout redirect: if the `subscriptionStatus` read fails, render normally (don't
  lock a paying tenant out on a transient error); log it.

## 6. Testing

- **Pure unit (TDD):** `isSuspended` (suspended set vs allowed/null);
  `statusForStripeInvoice` extended (`void`, `uncollectible`, `open`, `draft`, `paid`).
- **Mocked integration:** `/api/ingest` returns 403 for a suspended org;
  webhook records `org.suspended` on `active→canceled` and `org.reactivated` on
  `canceled→active` (assert the audit rows).
- **Manual:** in Stripe test mode, cancel a subscription → confirm ingest 403 +
  dashboard lock + audit row; re-activate → access restored.

## 7. File structure

| File | Responsibility | New? |
|---|---|---|
| `lib/billing/billing-status.ts` | + `isSuspended`, extend `statusForStripeInvoice` | Modify |
| `lib/db/schema.ts` | + `auditLog` table; widen `invoice.status` enum | Modify |
| `lib/audit.ts` | `recordAudit` + action constants | Create |
| `app/api/ingest/route.ts` | suspension 403 gate | Modify |
| `app/api/stripe/webhook/route.ts` | status-change + invoice + suspend/reactivate audit; overdue mapping | Modify |
| `middleware.ts` | inject `x-pathname` header | Modify |
| `app/(tenant)/layout.tsx` | suspended → lock to billing; past_due banner | Modify |
| `lib/actions/*.ts`, `app/(tenant)/.../actions.ts` | `recordAudit` calls | Modify |
| `lib/data.ts` | `getOrgAuditLog(orgId, limit)` | Modify |
| `app/(tenant)/tenant/activity/page.tsx` | tenant activity view | Create |
| `app/(admin)/admin/customers/[tenantId]/page.tsx` | admin "Activity" section | Modify |
| `lib/nav.ts` | + Activity in `TENANT_NAV` | Modify |

## 8. Sequencing

1. Pure helpers (`isSuspended`, status mapping) + tests.
2. `auditLog` schema + migration; `lib/audit.ts` writer.
3. Ingest suspension gate.
4. Middleware header + layout lock/banner.
5. Webhook: status-change/invoice/suspend audit + overdue mapping.
6. `recordAudit` calls across server actions.
7. Views: `getOrgAuditLog`, tenant `/tenant/activity` + nav, admin Activity section.
8. Manual Stripe-test-mode verification.
