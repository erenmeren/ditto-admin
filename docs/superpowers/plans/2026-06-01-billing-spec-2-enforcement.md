# Billing Spec 2 — Enforcement, Lifecycle & Audit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Suspend orgs whose Stripe subscription goes terminally unpaid (block ingest + lock dashboard), add `overdue`/`void` invoice states, and record notable account/billing/operational events to a new audit log viewable by platform admins and tenants.

**Architecture:** Suspension is **derived** from the webhook-maintained `tenantSettings.subscriptionStatus` (no new column) via a pure `isSuspended` helper. A best-effort `recordAudit` writer logs events from the webhook (system/stripe actor) and server actions (user actor). Pure logic is TDD'd without Stripe/DB.

**Tech Stack:** Next.js 16 App Router, Drizzle/Neon, Stripe webhooks, vitest.

---

## File Structure

| File | Responsibility | New? |
|---|---|---|
| `lib/billing/billing-status.ts` | + `isSuspended`; extend `statusForStripeInvoice` | Modify |
| `lib/billing/billing-status.test.ts` | tests for both | Modify |
| `lib/db/schema.ts` | + `auditLog` table; widen `invoice.status` enum; import `jsonb` | Modify |
| `lib/audit.ts` | `recordAudit` + `AUDIT` action constants | Create |
| `app/api/ingest/route.ts` | 403 when suspended (consolidate the settings read) | Modify |
| `middleware.ts` | inject `x-pathname` request header | Modify |
| `app/(tenant)/layout.tsx` | suspended → redirect to billing; past_due banner | Modify |
| `app/api/stripe/webhook/route.ts` | audit + overdue mapping + suspend/reactivate | Modify |
| `lib/actions/devices.ts`, `lib/actions/stores.ts`, `lib/actions/customers.ts`, `lib/actions/register.ts`, `app/(tenant)/tenant/branding/actions.ts`, `app/(tenant)/tenant/stores/[storeId]/actions.ts`, `app/(tenant)/tenant/billing/actions.ts` | `recordAudit` calls | Modify |
| `lib/data.ts` | `getOrgAuditLog(orgId, limit)` | Modify |
| `app/(tenant)/tenant/activity/page.tsx` | tenant activity view | Create |
| `app/(admin)/admin/customers/[tenantId]/page.tsx` | admin "Activity" section | Modify |
| `lib/nav.ts` | + Activity in `TENANT_NAV` | Modify |

> Manual Stripe-test-mode verification (cancel sub → 403 + lock + audit row) is the human's final step.

---

## Task 1: Pure helpers — `isSuspended` + extended status mapping (TDD)

**Files:** Modify `lib/billing/billing-status.ts`, `lib/billing/billing-status.test.ts`.

- [ ] **Step 1: Add failing tests** — append to `lib/billing/billing-status.test.ts` (create the file's imports if it doesn't import these yet; it currently tests `statusForStripeInvoice`/`meterEventPayload`). Add:

```ts
import { isSuspended } from "./billing-status";

describe("isSuspended", () => {
  it("suspends on canceled/unpaid/incomplete_expired", () => {
    expect(isSuspended("canceled")).toBe(true);
    expect(isSuspended("unpaid")).toBe(true);
    expect(isSuspended("incomplete_expired")).toBe(true);
  });
  it("allows null/active/past_due/trialing", () => {
    expect(isSuspended(null)).toBe(false);
    expect(isSuspended("active")).toBe(false);
    expect(isSuspended("past_due")).toBe(false);
    expect(isSuspended("trialing")).toBe(false);
  });
});

describe("statusForStripeInvoice extended", () => {
  it("maps void → void and uncollectible → overdue", () => {
    expect(statusForStripeInvoice("void")).toBe("void");
    expect(statusForStripeInvoice("uncollectible")).toBe("overdue");
  });
});
```

(The existing `statusForStripeInvoice` import at the top of the test file already covers the second block. If `statusForStripeInvoice` is not imported in that file, add it to the existing import.)

- [ ] **Step 2: Run, expect FAIL**

Run: `npm test -- lib/billing/billing-status.test.ts`
Expected: FAIL (`isSuspended` not exported; `void`/`uncollectible` cases wrong).

- [ ] **Step 3: Implement** in `lib/billing/billing-status.ts`. Change the `InvoiceStatus` type and `statusForStripeInvoice`, and add `isSuspended`:

```ts
type InvoiceStatus = "draft" | "sent" | "paid" | "overdue" | "void";

/** Map a Stripe invoice status onto our enum. */
export function statusForStripeInvoice(stripeStatus: string): InvoiceStatus {
  switch (stripeStatus) {
    case "draft":
      return "draft";
    case "paid":
      return "paid";
    case "void":
      return "void";
    case "uncollectible":
      return "overdue";
    default:
      return "sent"; // open
  }
}

const SUSPENDED_STATUSES = new Set(["canceled", "unpaid", "incomplete_expired"]);

/** True when a subscription is terminally unpaid (org should be suspended). */
export function isSuspended(subscriptionStatus: string | null): boolean {
  return subscriptionStatus != null && SUSPENDED_STATUSES.has(subscriptionStatus);
}
```

- [ ] **Step 4: Run, expect PASS**

Run: `npm test -- lib/billing/billing-status.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/billing/billing-status.ts lib/billing/billing-status.test.ts
git commit -m "feat: add isSuspended + overdue/void invoice status mapping"
```

---

## Task 2: `auditLog` table + widen invoice enum + migration

**Files:** Modify `lib/db/schema.ts`.

- [ ] **Step 1: Import `jsonb`** — in the `drizzle-orm/pg-core` import block, add `jsonb`:

```ts
import {
  boolean,
  index,
  integer,
  jsonb,
  pgTable,
  text,
  timestamp,
  uniqueIndex,
} from "drizzle-orm/pg-core";
```

- [ ] **Step 2: Widen the invoice status enum** — change the `invoice.status` column enum to include the new values:

```ts
    status: text("status", { enum: ["draft", "sent", "paid", "overdue", "void"] })
      .default("draft")
      .notNull(),
```

- [ ] **Step 3: Add the `auditLog` table** — after the `invoice` table definition (before the `schema` re-export map):

```ts
export const auditLog = pgTable(
  "audit_log",
  {
    id: text("id").primaryKey(),
    organizationId: text("organization_id")
      .notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    actorType: text("actor_type", { enum: ["user", "system", "stripe"] }).notNull(),
    actorId: text("actor_id"),
    actorLabel: text("actor_label"),
    action: text("action").notNull(),
    targetType: text("target_type"),
    targetId: text("target_id"),
    metadata: jsonb("metadata"),
    createdAt: timestamp("created_at")
      .$defaultFn(() => new Date())
      .notNull(),
  },
  (t) => [index("audit_log_org_created_idx").on(t.organizationId, t.createdAt)],
);
```

- [ ] **Step 4: Add `auditLog` to the `schema` map** — in the `export const schema = { ... }` object, add `auditLog,` alongside the other tables.

- [ ] **Step 5: Generate + apply the migration**

Run: `npm run db:generate`
Expected: a new SQL file creating `audit_log` + its index. The `invoice.status` enum widening produces **no** SQL (DB column is plain `text`). Inspect: only `CREATE TABLE "audit_log"` + `CREATE INDEX`, no destructive statements.

Run: `npm run db:migrate`
Expected: applies cleanly.

- [ ] **Step 6: Verify + commit**

Run: `npx tsc --noEmit` (expect clean)

```bash
git add lib/db/schema.ts drizzle/
git commit -m "feat: add audit_log table and widen invoice status enum"
```

---

## Task 3: `recordAudit` writer + action constants

**Files:** Create `lib/audit.ts`.

- [ ] **Step 1: Write the module**

```ts
// lib/audit.ts
// Best-effort audit logging. recordAudit never throws into its caller — auditing
// a device delete must not be able to fail the delete.

import { db } from "@/lib/db";
import { auditLog } from "@/lib/db/schema";
import { id } from "@/lib/ids";

export type AuditActor =
  | { type: "user"; id: string; label: string }
  | { type: "system" }
  | { type: "stripe" };

/** Action name constants (stringly-typed at the DB; centralized here). */
export const AUDIT = {
  orgCreated: "org.created",
  orgSuspended: "org.suspended",
  orgReactivated: "org.reactivated",
  subscriptionStatusChanged: "subscription.status_changed",
  invoicePaid: "invoice.paid",
  invoicePaymentFailed: "invoice.payment_failed",
  invoiceVoid: "invoice.void",
  billingActivated: "billing.activated",
  customerCreated: "customer.created",
  deviceProvisioned: "device.provisioned",
  deviceRenamed: "device.renamed",
  deviceReassigned: "device.reassigned",
  deviceDeleted: "device.deleted",
  devicePaused: "device.paused",
  deviceResumed: "device.resumed",
  deviceClaimed: "device.claimed",
  storeCreated: "store.created",
  brandingUpdated: "branding.updated",
} as const;

export async function recordAudit(input: {
  organizationId: string;
  actor: AuditActor;
  action: string;
  target?: { type: string; id: string };
  metadata?: Record<string, unknown>;
}): Promise<void> {
  try {
    await db.insert(auditLog).values({
      id: id("aud"),
      organizationId: input.organizationId,
      actorType: input.actor.type,
      actorId: input.actor.type === "user" ? input.actor.id : null,
      actorLabel: input.actor.type === "user" ? input.actor.label : input.actor.type,
      action: input.action,
      targetType: input.target?.type ?? null,
      targetId: input.target?.id ?? null,
      metadata: input.metadata ?? null,
    });
  } catch (err) {
    console.error("[audit] failed to record", input.action, err);
  }
}
```

- [ ] **Step 2: Verify type-check**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add lib/audit.ts
git commit -m "feat: add best-effort audit log writer"
```

---

## Task 4: Ingest suspension gate

**Files:** Modify `app/api/ingest/route.ts`.

The route already fetches `tenantSettings` (for metering) **after** building the receipt. Add an **early** suspension check after device auth, selecting `subscriptionStatus`, and fail *safe* on a DB error.

- [ ] **Step 1: Add the import** to the existing `@/lib/billing/...` imports:

```ts
import { isSuspended } from "@/lib/billing/billing-status";
```

- [ ] **Step 2: Insert the gate** immediately after the existing `if (device.status === "paused") return bad(403, "Device is paused");` line:

```ts
  // Block ingestion for orgs whose subscription is terminally unpaid. Fail safe
  // (allow) on a transient read error so a DB blip can't block a paying customer.
  try {
    const [billing] = await db
      .select({ status: tenantSettings.subscriptionStatus })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, device.organizationId))
      .limit(1);
    if (isSuspended(billing?.status ?? null)) {
      return bad(403, "Subscription inactive");
    }
  } catch (err) {
    console.error("[ingest] suspension check failed (allowing)", err);
  }
```

(`tenantSettings` is already imported from `@/lib/db/schema` after Spec 1's Task 8.)

- [ ] **Step 3: Verify**

Run: `npx tsc --noEmit && npm test`
Expected: no type errors; all tests pass.

- [ ] **Step 4: Commit**

```bash
git add app/api/ingest/route.ts
git commit -m "feat: block ingest for suspended orgs"
```

---

## Task 5: Middleware `x-pathname` + tenant dashboard lock/banner

**Files:** Modify `middleware.ts`, `app/(tenant)/layout.tsx`.

- [ ] **Step 1: Inject `x-pathname` in middleware** — replace the `return NextResponse.next();` line (the authenticated branch) with a version that forwards the path as a request header:

```ts
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-pathname", request.nextUrl.pathname);
  return NextResponse.next({ request: { headers: requestHeaders } });
```

- [ ] **Step 2: Lock the tenant layout** — rewrite `app/(tenant)/layout.tsx`:

```tsx
import { headers } from "next/headers";
import { redirect } from "next/navigation";
import { eq } from "drizzle-orm";
import { AppShell } from "@/components/app-shell";
import { requireTenant } from "@/lib/session";
import { db } from "@/lib/db";
import { tenantSettings } from "@/lib/db/schema";
import { isSuspended } from "@/lib/billing/billing-status";

export default async function TenantLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const { ctx, organizationId } = await requireTenant();
  const activeName =
    ctx.organizations.find((o) => o.id === organizationId)?.name ?? "Workspace";

  // Billing enforcement. Fail safe (no lock) on a transient read error.
  let subStatus: string | null = null;
  try {
    const [s] = await db
      .select({ status: tenantSettings.subscriptionStatus })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, organizationId))
      .limit(1);
    subStatus = s?.status ?? null;
  } catch (err) {
    console.error("[tenant layout] billing status read failed", err);
  }

  const pathname = (await headers()).get("x-pathname") ?? "";
  if (isSuspended(subStatus) && pathname !== "/tenant/billing") {
    redirect("/tenant/billing");
  }
  const pastDue = subStatus === "past_due";

  return (
    <AppShell
      workspace="tenant"
      groupLabel="Workspace"
      topBarLabel={activeName}
      user={ctx.user}
      organizations={ctx.organizations}
      role={ctx.user.role}
      activeName={activeName}
      activeOrganizationId={organizationId}
    >
      {pastDue ? (
        <div className="mb-4 rounded-lg border border-amber-300 bg-amber-50 px-4 py-3 text-sm text-amber-900 dark:border-amber-900/50 dark:bg-amber-950/40 dark:text-amber-200">
          Your last payment failed. Update your payment method on the{" "}
          <a href="/tenant/billing" className="font-medium underline">
            Billing
          </a>{" "}
          page to avoid interruption.
        </div>
      ) : null}
      {children}
    </AppShell>
  );
}
```

- [ ] **Step 3: Verify**

Run: `npx tsc --noEmit`
Expected: no type errors.

- [ ] **Step 4: Commit**

```bash
git add middleware.ts "app/(tenant)/layout.tsx"
git commit -m "feat: lock suspended tenants to billing; past_due banner"
```

---

## Task 6: Webhook — audit events + overdue mapping + suspend/reactivate

**Files:** Modify `app/api/stripe/webhook/route.ts`.

- [ ] **Step 1: Add imports** near the top:

```ts
import { invoice as invoiceTable } from "@/lib/db/schema";
import { isSuspended } from "@/lib/billing/billing-status";
import { recordAudit, AUDIT } from "@/lib/audit";
```

(`tenantSettings` and `eq` are already imported.)

- [ ] **Step 2: Replace the subscription case** to compare prev→new, persist, and audit transitions:

```ts
    case "customer.subscription.created":
    case "customer.subscription.updated":
    case "customer.subscription.deleted": {
      const sub = event.data.object as Stripe.Subscription;
      const customerId = typeof sub.customer === "string" ? sub.customer : sub.customer.id;
      const [row] = await db
        .select({ org: tenantSettings.organizationId, prev: tenantSettings.subscriptionStatus })
        .from(tenantSettings)
        .where(eq(tenantSettings.stripeCustomerId, customerId))
        .limit(1);
      if (!row) break;

      await db
        .update(tenantSettings)
        .set({ stripeSubscriptionId: sub.id, subscriptionStatus: sub.status })
        .where(eq(tenantSettings.organizationId, row.org));

      if (row.prev !== sub.status) {
        await recordAudit({
          organizationId: row.org,
          actor: { type: "stripe" },
          action: AUDIT.subscriptionStatusChanged,
          metadata: { from: row.prev, to: sub.status },
        });
        const was = isSuspended(row.prev ?? null);
        const now = isSuspended(sub.status);
        if (!was && now) {
          await recordAudit({ organizationId: row.org, actor: { type: "stripe" }, action: AUDIT.orgSuspended, metadata: { status: sub.status } });
        } else if (was && !now) {
          await recordAudit({ organizationId: row.org, actor: { type: "stripe" }, action: AUDIT.orgReactivated, metadata: { status: sub.status } });
        }
      }
      break;
    }
```

- [ ] **Step 3: Replace the invoice case** to audit + apply the `overdue` override on failure:

```ts
    case "invoice.created":
    case "invoice.finalized":
    case "invoice.paid":
    case "invoice.payment_failed":
    case "invoice.voided": {
      const si = event.data.object as Stripe.Invoice;
      await upsertInvoiceFromStripe(si);

      const customerId = typeof si.customer === "string" ? si.customer : si.customer?.id;
      if (customerId && si.id) {
        const [row] = await db
          .select({ org: tenantSettings.organizationId })
          .from(tenantSettings)
          .where(eq(tenantSettings.stripeCustomerId, customerId))
          .limit(1);
        if (row) {
          if (event.type === "invoice.payment_failed") {
            await db.update(invoiceTable).set({ status: "overdue" }).where(eq(invoiceTable.stripeInvoiceId, si.id));
            await recordAudit({ organizationId: row.org, actor: { type: "stripe" }, action: AUDIT.invoicePaymentFailed, target: { type: "invoice", id: si.id }, metadata: { amountDueCents: si.amount_due } });
          } else if (event.type === "invoice.paid") {
            await recordAudit({ organizationId: row.org, actor: { type: "stripe" }, action: AUDIT.invoicePaid, target: { type: "invoice", id: si.id }, metadata: { amountDueCents: si.amount_due } });
          } else if (event.type === "invoice.voided") {
            await recordAudit({ organizationId: row.org, actor: { type: "stripe" }, action: AUDIT.invoiceVoid, target: { type: "invoice", id: si.id } });
          }
        }
      }
      break;
    }
```

(Remove the old separate `case "invoice.created": ... case "invoice.payment_failed":` block that only called `upsertInvoiceFromStripe` — this replaces it.)

- [ ] **Step 4: Verify**

Run: `npx tsc --noEmit && npm test`
Expected: no type errors; tests pass.

- [ ] **Step 5: Commit**

```bash
git add app/api/stripe/webhook/route.ts
git commit -m "feat: audit billing events and mark overdue invoices in webhook"
```

---

## Task 7: `recordAudit` calls across server actions

Add one `recordAudit` call at each successful mutation. Actor for tenant actions:
`{ type: "user", id: ctx.user.id, label: ctx.user.email }` (get `ctx` from
`requireTenant()` / `requirePlatformAdmin()` — note: device actions currently
destructure only `organizationId` from `requireTenant()`; change to
`const { ctx, organizationId } = await requireTenant();`).

**Files:** Modify `lib/actions/devices.ts`, `lib/actions/stores.ts`, `lib/actions/customers.ts`, `lib/actions/register.ts`, `app/(tenant)/tenant/branding/actions.ts`, `app/(tenant)/tenant/stores/[storeId]/actions.ts`, `app/(tenant)/tenant/billing/actions.ts`.

- [ ] **Step 1: Import in each file**

```ts
import { recordAudit, AUDIT } from "@/lib/audit";
```

- [ ] **Step 2: `lib/actions/devices.ts`** — after each successful mutation, before the success `return`, add the matching audit call. For `setDeviceActive` (pause/resume), use the resulting status:

```ts
  await recordAudit({
    organizationId,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: next === "paused" ? AUDIT.devicePaused : AUDIT.deviceResumed,
    target: { type: "device", id: device.id },
  });
```

For rename / reassign / delete / provision, use `AUDIT.deviceRenamed` / `AUDIT.deviceReassigned` / `AUDIT.deviceDeleted` / `AUDIT.deviceProvisioned` with `target: { type: "device", id: <deviceId> }` and relevant `metadata` (e.g. `{ name }` for rename, `{ storeId }` for reassign). Change each handler's `requireTenant()`/`requirePlatformAdmin()` destructure to also capture `ctx`.

- [ ] **Step 3: `lib/actions/stores.ts`** (store create) — after insert:

```ts
  await recordAudit({
    organizationId,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.storeCreated,
    target: { type: "store", id: store.id },
    metadata: { name: store.name },
  });
```

- [ ] **Step 4: `app/(tenant)/tenant/stores/[storeId]/actions.ts`** (device claim) — after a successful claim:

```ts
  await recordAudit({
    organizationId,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.deviceClaimed,
    target: { type: "device", id: deviceId },
  });
```

- [ ] **Step 5: `app/(tenant)/tenant/branding/actions.ts`** — after the settings update:

```ts
  await recordAudit({
    organizationId,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.brandingUpdated,
  });
```

- [ ] **Step 6: `app/(tenant)/tenant/billing/actions.ts`** — after `activate(organizationId)` succeeds, before returning:

```ts
  await recordAudit({
    organizationId,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.billingActivated,
  });
```

(Change the destructure to `const { ctx, organizationId } = await requireTenant();`.)

- [ ] **Step 7: `lib/actions/customers.ts`** (admin creates a customer/org) — use `actor` from `requirePlatformAdmin()` (`ctx.user`) and the created org id:

```ts
  await recordAudit({
    organizationId: org.id,
    actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.customerCreated,
    metadata: { name: org.name },
  });
```

- [ ] **Step 8: `lib/actions/register.ts`** (self-serve signup) — after the org + owner are created, record `org.created` with a `system`-flavored user actor (the new owner):

```ts
  await recordAudit({
    organizationId: newOrgId,
    actor: { type: "user", id: newUserId, label: email },
    action: AUDIT.orgCreated,
    metadata: { name: companyName },
  });
```

Use whatever the file already names the created org id / user id / company name / email variables (read the file; adapt names).

- [ ] **Step 9: Verify**

Run: `npx tsc --noEmit && npm test`
Expected: no type errors; tests pass.

- [ ] **Step 10: Commit**

```bash
git add lib/actions/ "app/(tenant)/tenant/branding/actions.ts" "app/(tenant)/tenant/stores/[storeId]/actions.ts" "app/(tenant)/tenant/billing/actions.ts"
git commit -m "feat: record audit events from server actions"
```

---

## Task 8: Views — data fn + tenant activity page + admin section + nav

**Files:** Modify `lib/data.ts`, `lib/nav.ts`, `app/(admin)/admin/customers/[tenantId]/page.tsx`; Create `app/(tenant)/tenant/activity/page.tsx`.

- [ ] **Step 1: Add `getOrgAuditLog` to `lib/data.ts`** (match the file's existing import style for `db`/`eq`/`desc`/table aliases; add `auditLog` import):

```ts
export async function getOrgAuditLog(organizationId: string, limit = 100) {
  const rows = await db
    .select()
    .from(auditLog)
    .where(eq(auditLog.organizationId, organizationId))
    .orderBy(desc(auditLog.createdAt))
    .limit(limit);
  return rows.map((r) => ({
    id: r.id,
    action: r.action,
    actor: r.actorLabel ?? r.actorType,
    target: r.targetType ? `${r.targetType}:${r.targetId}` : null,
    metadata: (r.metadata as Record<string, unknown> | null) ?? null,
    at: r.createdAt.toISOString(),
  }));
}
```

- [ ] **Step 2: Create the tenant activity page** `app/(tenant)/tenant/activity/page.tsx`:

```tsx
import { requireTenant } from "@/lib/session";
import { getOrgAuditLog } from "@/lib/data";

export default async function ActivityPage() {
  const { organizationId } = await requireTenant();
  const events = await getOrgAuditLog(organizationId);

  return (
    <div className="flex flex-col gap-6 p-6">
      <h1 className="text-2xl font-semibold tracking-tight">Activity</h1>
      {events.length === 0 ? (
        <p className="text-sm text-muted-foreground">No activity yet.</p>
      ) : (
        <table className="w-full text-sm">
          <thead>
            <tr className="text-left text-muted-foreground">
              <th className="py-2">When</th>
              <th>Action</th>
              <th>By</th>
            </tr>
          </thead>
          <tbody>
            {events.map((e) => (
              <tr key={e.id} className="border-t">
                <td className="py-2">{e.at.slice(0, 19).replace("T", " ")}</td>
                <td>{e.action}</td>
                <td>{e.actor}</td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Add Activity to `TENANT_NAV`** in `lib/nav.ts` — import an icon (`Activity` from lucide-react) and add after Billing:

```ts
  { label: "Activity", href: "/tenant/activity", icon: Activity },
```

(Add `Activity` to the lucide-react import.)

- [ ] **Step 4: Admin "Activity" section** — in `app/(admin)/admin/customers/[tenantId]/page.tsx`, import `getOrgAuditLog`, fetch `const activity = await getOrgAuditLog(tenantId, 50);` (use the page's existing org-id param name), and render a section below the existing content:

```tsx
      <section className="flex flex-col gap-3">
        <h2 className="text-lg font-medium">Activity</h2>
        {activity.length === 0 ? (
          <p className="text-sm text-muted-foreground">No activity yet.</p>
        ) : (
          <ul className="flex flex-col gap-1 text-sm">
            {activity.map((e) => (
              <li key={e.id} className="flex justify-between border-t py-1.5">
                <span>{e.action}</span>
                <span className="text-muted-foreground">
                  {e.actor} · {e.at.slice(0, 19).replace("T", " ")}
                </span>
              </li>
            ))}
          </ul>
        )}
      </section>
```

(Match the param name the page uses for the tenant/org id — read the file; it may be `params.tenantId`.)

- [ ] **Step 5: Verify**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors; build succeeds with `/tenant/activity` listed.

- [ ] **Step 6: Commit**

```bash
git add lib/data.ts lib/nav.ts "app/(tenant)/tenant/activity/" "app/(admin)/admin/customers/[tenantId]/page.tsx"
git commit -m "feat: audit log views for tenant and platform admin"
```

---

## Task 9: Manual verification (Stripe test mode — human-run)

- [ ] In Stripe test mode, cancel the org's subscription (Dashboard or API).
- [ ] Confirm the `customer.subscription.updated`/`deleted` webhook sets
      `subscriptionStatus` and writes `org.suspended` (check `audit_log` via `npm run db:studio`).
- [ ] `POST /api/ingest` with that org's device key → expect `403`.
- [ ] Visit `/tenant` as that org → redirected to `/tenant/billing`.
- [ ] Re-activate billing → `subscriptionStatus` active → ingest + dashboard restored; `org.reactivated` row present.
- [ ] Trigger a failed payment (test card `4000 0000 0000 0341`) → invoice row flips to `overdue`, `invoice.payment_failed` audit row present, banner shows for `past_due`.

---

## Self-Review

- **Spec coverage:** isSuspended + status mapping (T1); auditLog schema + enum (T2); recordAudit (T3); ingest 403 (T4); middleware + layout lock/banner (T5); webhook audit + overdue + suspend/reactivate (T6); server-action audit (T7); views + nav (T8); manual verify (T9). All spec sections mapped.
- **Placeholder scan:** code-complete steps; Tasks 7–8 say "match the file's existing variable/param names" where I can't see exact local identifiers — these are adaptation notes, not missing logic (the audit call shape is fully specified).
- **Type consistency:** `isSuspended` (T1) used in T4/T5/T6; `recordAudit`/`AUDIT` (T3) used in T6/T7; `auditLog` table (T2) used in T3/T8; `getOrgAuditLog` (T8) used by both views; `InvoiceStatus` widened (T1) matches the schema enum (T2) and the `overdue` write (T6).

## Execution notes

- **Runs green now:** Tasks 1, 3 (pure/standalone). T2 migration needs `DATABASE_URL`.
- **No Stripe keys needed** for any code task; T9 needs Stripe test mode (human).
