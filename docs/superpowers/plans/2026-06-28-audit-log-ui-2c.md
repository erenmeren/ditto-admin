# Phase 2C — Audit-Log UI Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the audit log readable and paginable — human-readable action labels, a target column, `timeAgo` timestamps, an actor-type chip, and offset pagination on the tenant Activity page.

**Architecture:** A pure `actionLabel`/`humanizeAction` module; a new `getOrgAuditPage` offset-paginated reader + an `actorType` field on the existing `getOrgAuditLog`; the tenant Activity page rebuilt with the shadcn `Table` + `?page=N` pagination (mirroring the documents pattern); and a light label/target/timeAgo touch on the admin customer-detail audit section.

**Tech Stack:** Next.js 16 RSC, Drizzle/Neon (`count` already imported in `lib/data.ts`), Vitest (pure labels). Reuses `components/ui/table.tsx`, `timeAgo` (`lib/format.ts`), and the documents `?page=N` link pattern.

## Global Constraints

- **Labels:** a map of all 37 `AUDIT.*` action strings → friendly labels, with a `humanizeAction` fallback for any unmapped/future action. `actionLabel(a) = AUDIT_LABELS[a] ?? humanizeAction(a)`.
- **Pagination:** offset/page-number, **page size 25**, tenant Activity page only (`?page=N`, mirroring `components/documents/documents-table.tsx`). Admin section stays a recent-50 summary.
- **Show `target`; do NOT render `metadata`.**
- **Actor chip:** a small muted chip for `system`/`stripe` actors (`user` plain).
- A guard test asserts every value in `AUDIT` has an `AUDIT_LABELS` entry (map can't drift from constants).
- Verification per task: `npm run test` (currently 300, stays green), `npm run build`, `npx tsc --noEmit`. Dev server on **:3001**.

## File Map

| File | Responsibility | Task |
|---|---|---|
| `lib/audit-labels.ts` (new) + `.test.ts` | pure `AUDIT_LABELS` + `humanizeAction` + `actionLabel` | 1 |
| `lib/data.ts` (`getOrgAuditLog` + new `getOrgAuditPage`) | `actorType` field + paged reader | 2 |
| `app/(tenant)/tenant/activity/page.tsx` (rewrite) | shadcn Table + labels + chip + target + `?page` | 3 |
| `app/(admin)/admin/customers/[tenantId]/page.tsx` (modify) | labels + target + timeAgo in the audit `<ul>` | 4 |

---

### Task 1: Pure audit label module

**Files:**
- Create: `lib/audit-labels.ts`
- Test: `lib/audit-labels.test.ts`

**Interfaces:**
- Consumes: `AUDIT` (`lib/audit.ts`) — only in the test, for the completeness guard.
- Produces: `AUDIT_LABELS: Record<string, string>`; `humanizeAction(action: string): string`; `actionLabel(action: string): string`.

- [ ] **Step 1: Write the failing test**

Create `lib/audit-labels.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { AUDIT_LABELS, humanizeAction, actionLabel } from "./audit-labels";
import { AUDIT } from "./audit";

describe("actionLabel", () => {
  it("maps known actions to friendly labels", () => {
    expect(actionLabel("device.went_offline")).toBe("Device went offline");
    expect(actionLabel("invoice.sent")).toBe("Invoice sent");
    expect(actionLabel("credits.purchased")).toBe("Credits purchased");
    expect(actionLabel("invoice.payment_failed")).toBe("Payment failed");
  });
  it("falls back to humanizeAction for an unmapped action", () => {
    expect(actionLabel("foo.bar_baz")).toBe("Foo: Bar baz");
  });
});

describe("humanizeAction", () => {
  it("title-cases entity + verb, underscores → spaces", () => {
    expect(humanizeAction("device.command_enqueued")).toBe("Device: Command enqueued");
    expect(humanizeAction("api_key.revoked")).toBe("Api key: Revoked");
  });
  it("handles a bare entity with no verb", () => {
    expect(humanizeAction("created")).toBe("Created");
  });
});

describe("AUDIT_LABELS completeness", () => {
  it("has a label for every AUDIT constant (map cannot drift)", () => {
    for (const value of Object.values(AUDIT)) {
      expect(AUDIT_LABELS[value], `missing label for "${value}"`).toBeDefined();
    }
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run lib/audit-labels.test.ts`
Expected: FAIL — `Cannot find module './audit-labels'`.

- [ ] **Step 3: Write the implementation**

Create `lib/audit-labels.ts`:

```ts
// lib/audit-labels.ts
// Pure: turn raw audit action strings (lib/audit.ts AUDIT.*) into human-readable
// labels. Every known action is mapped; humanizeAction() is the safe fallback for
// any future/unmapped string. No IO.

export const AUDIT_LABELS: Record<string, string> = {
  "org.created": "Organization created",
  "org.suspended": "Organization suspended",
  "org.reactivated": "Organization reactivated",
  "subscription.status_changed": "Subscription status changed",
  "invoice.paid": "Invoice paid",
  "invoice.payment_failed": "Payment failed",
  "invoice.void": "Invoice voided",
  "invoice.sent": "Invoice sent",
  "invoice.overdue": "Invoice overdue",
  "billing.activated": "Billing activated",
  "customer.created": "Customer created",
  "device.provisioned": "Device provisioned",
  "device.renamed": "Device renamed",
  "device.reassigned": "Device reassigned",
  "device.unassigned": "Device unassigned",
  "device.command_enqueued": "Command sent to device",
  "device.deleted": "Device deleted",
  "device.paused": "Device paused",
  "device.resumed": "Device resumed",
  "device.claimed": "Device claimed",
  "device.went_offline": "Device went offline",
  "store.created": "Store created",
  "store.updated": "Store updated",
  "api_key.created": "API key created",
  "api_key.revoked": "API key revoked",
  "webhook_endpoint.created": "Webhook endpoint created",
  "webhook_endpoint.deleted": "Webhook endpoint deleted",
  "webhook_endpoint.disabled": "Webhook endpoint disabled",
  "branding.updated": "Branding updated",
  "device_settings.updated": "Device settings updated",
  "member.invited": "Member invited",
  "member.added": "Member added",
  "member.removed": "Member removed",
  "member.role_changed": "Member role changed",
  "invitation.canceled": "Invitation canceled",
  "credits.granted": "Credits granted",
  "credits.purchased": "Credits purchased",
};

function cap(s: string): string {
  return s ? s[0].toUpperCase() + s.slice(1) : s;
}

/** Fallback for any action not in AUDIT_LABELS: "device.command_enqueued" → "Device: Command enqueued". */
export function humanizeAction(action: string): string {
  const dot = action.indexOf(".");
  const entity = (dot === -1 ? action : action.slice(0, dot)).replace(/_/g, " ");
  const verb = (dot === -1 ? "" : action.slice(dot + 1)).replace(/_/g, " ");
  return verb ? `${cap(entity)}: ${cap(verb)}` : cap(entity);
}

export function actionLabel(action: string): string {
  return AUDIT_LABELS[action] ?? humanizeAction(action);
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run lib/audit-labels.test.ts`
Expected: PASS (all cases, incl. the completeness guard over all 37 `AUDIT` values).

- [ ] **Step 5: Full suite + typecheck**

Run: `npm run test && npx tsc --noEmit`
Expected: all green (300 + new), no type errors.

- [ ] **Step 6: Commit**

```bash
git add lib/audit-labels.ts lib/audit-labels.test.ts
git commit -m "feat(audit): pure action-label map + humanizer (2C)"
```

---

### Task 2: Audit data layer — actorType + paged reader

**Files:**
- Modify: `lib/data.ts` (`getOrgAuditLog`, new `getOrgAuditPage`)

**Interfaces:**
- Consumes: `auditLogTable` (existing alias in `lib/data.ts`), `count`/`desc`/`eq` from drizzle (all already imported).
- Produces:
  - `getOrgAuditLog` rows gain `actorType: string`.
  - `getOrgAuditPage(organizationId: string, page: number, pageSize?: number): Promise<{ rows: AuditRow[]; total: number; page: number; pageSize: number; pageCount: number }>` where `AuditRow = { id: string; action: string; actorType: string; actor: string; target: string | null; metadata: Record<string, unknown> | null; at: string }`.

- [ ] **Step 1: Add `actorType` to `getOrgAuditLog`'s mapped rows**

In `lib/data.ts`, `getOrgAuditLog`'s `.map(...)` currently returns `{ id, action, actor, target, metadata, at }`. Add `actorType`:

```ts
  return rows.map((r) => ({
    id: r.id,
    action: r.action,
    actorType: r.actorType,
    actor: r.actorLabel ?? r.actorType,
    target: r.targetType && r.targetId ? `${r.targetType}:${r.targetId}` : null,
    metadata: (r.metadata as Record<string, unknown> | null) ?? null,
    at: r.createdAt.toISOString(),
  }));
```

- [ ] **Step 2: Add `getOrgAuditPage` right after `getOrgAuditLog`**

Append in `lib/data.ts` (uses the same `auditLogTable` alias, `count`, `desc`, `eq` — all already imported):

```ts
export async function getOrgAuditPage(
  organizationId: string,
  page: number,
  pageSize = 25,
): Promise<{
  rows: {
    id: string;
    action: string;
    actorType: string;
    actor: string;
    target: string | null;
    metadata: Record<string, unknown> | null;
    at: string;
  }[];
  total: number;
  page: number;
  pageSize: number;
  pageCount: number;
}> {
  const safePage = Math.max(1, Math.floor(page) || 1);
  const [{ total }] = await db
    .select({ total: count() })
    .from(auditLogTable)
    .where(eq(auditLogTable.organizationId, organizationId));
  const pageCount = Math.max(1, Math.ceil(total / pageSize));

  const rows = await db
    .select()
    .from(auditLogTable)
    .where(eq(auditLogTable.organizationId, organizationId))
    .orderBy(desc(auditLogTable.createdAt))
    .limit(pageSize)
    .offset((safePage - 1) * pageSize);

  return {
    rows: rows.map((r) => ({
      id: r.id,
      action: r.action,
      actorType: r.actorType,
      actor: r.actorLabel ?? r.actorType,
      target: r.targetType && r.targetId ? `${r.targetType}:${r.targetId}` : null,
      metadata: (r.metadata as Record<string, unknown> | null) ?? null,
      at: r.createdAt.toISOString(),
    })),
    total,
    page: safePage,
    pageSize,
    pageCount,
  };
}
```

- [ ] **Step 3: Typecheck + build + suite**

Run: `npx tsc --noEmit && npm run build && npm run test`
Expected: no type errors; build OK; 300+ green (adding `actorType` is additive; existing tenant/admin pages still compile since they read named fields).

- [ ] **Step 4: Commit**

```bash
git add lib/data.ts
git commit -m "feat(audit): actorType field + getOrgAuditPage offset reader (2C)"
```

---

### Task 3: Tenant Activity page — table + pagination

**Files:**
- Modify (rewrite): `app/(tenant)/tenant/activity/page.tsx`

**Interfaces:**
- Consumes: `getOrgAuditPage` (Task 2); `actionLabel` (Task 1); `timeAgo` (`lib/format.ts`); the shadcn `Table` primitives (`components/ui/table.tsx`); `requireTenant`.

- [ ] **Step 1: Rewrite the page**

Replace `app/(tenant)/tenant/activity/page.tsx` entirely with:

```tsx
import Link from "next/link";
import { requireTenant } from "@/lib/session";
import { getOrgAuditPage } from "@/lib/data";
import { actionLabel } from "@/lib/audit-labels";
import { timeAgo } from "@/lib/format";
import { Card } from "@/components/ui/card";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

export default async function ActivityPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | undefined>>;
}) {
  const { organizationId } = await requireTenant();
  const raw = await searchParams;
  const requested = Math.max(1, Number(raw.page) || 1);
  const { rows, page, pageCount } = await getOrgAuditPage(organizationId, requested);

  return (
    <div className="flex flex-col gap-6 p-6">
      <h1 className="text-2xl font-semibold tracking-tight">Activity</h1>

      <Card className="overflow-hidden py-0">
        <Table>
          <TableHeader>
            <TableRow className="hover:bg-transparent">
              <TableHead className="pl-6">When</TableHead>
              <TableHead>Action</TableHead>
              <TableHead>Actor</TableHead>
              <TableHead>Target</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {rows.map((e) => (
              <TableRow key={e.id}>
                <TableCell className="pl-6 text-muted-foreground">{timeAgo(e.at)}</TableCell>
                <TableCell className="font-medium">{actionLabel(e.action)}</TableCell>
                <TableCell>
                  {e.actor}
                  {e.actorType !== "user" && (
                    <span className="ml-1.5 rounded bg-muted px-1.5 py-0.5 text-[10px] uppercase text-muted-foreground">
                      {e.actorType}
                    </span>
                  )}
                </TableCell>
                <TableCell className="font-mono text-xs text-muted-foreground">{e.target ?? "—"}</TableCell>
              </TableRow>
            ))}
            {rows.length === 0 && (
              <TableRow className="hover:bg-transparent">
                <TableCell colSpan={4} className="py-12 text-center text-sm text-muted-foreground">
                  No activity yet.
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </Card>

      <div className="flex items-center justify-between text-sm">
        <span className="text-muted-foreground">Page {page} of {pageCount}</span>
        <span className="flex gap-3">
          {page > 1 ? (
            <Link className="underline" href={`/tenant/activity?page=${page - 1}`}>Previous</Link>
          ) : (
            <span className="text-muted-foreground">Previous</span>
          )}
          {page < pageCount ? (
            <Link className="underline" href={`/tenant/activity?page=${page + 1}`}>Next</Link>
          ) : (
            <span className="text-muted-foreground">Next</span>
          )}
        </span>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors; build OK; the `/tenant/activity` route compiles.

- [ ] **Step 3: Commit**

```bash
git add "app/(tenant)/tenant/activity/page.tsx"
git commit -m "feat(audit): tenant Activity page — labels, actor chip, target, pagination (2C)"
```

---

### Task 4: Admin customer-detail audit section — labels + target + timeAgo

**Files:**
- Modify: `app/(admin)/admin/customers/[tenantId]/page.tsx`

**Interfaces:**
- Consumes: `actionLabel` (Task 1); `timeAgo` (already imported in this page); `getOrgAuditLog` rows (now incl. `actorType`).

- [ ] **Step 1: Import `actionLabel`**

In `app/(admin)/admin/customers/[tenantId]/page.tsx`, add to the imports:

```tsx
import { actionLabel } from "@/lib/audit-labels";
```

- [ ] **Step 2: Use the label + target + timeAgo in the audit `<ul>`**

The audit section's `<li>` currently is:

```tsx
              <li key={e.id} className="flex justify-between border-t py-1.5">
                <span>{e.action}</span>
                <span className="text-muted-foreground">
                  {e.actor} · {e.at.slice(0, 19).replace("T", " ")}
                </span>
              </li>
```

Replace it with (friendly label + target on the left; actor + `timeAgo` on the right):

```tsx
              <li key={e.id} className="flex justify-between gap-4 border-t py-1.5">
                <span>
                  {actionLabel(e.action)}
                  {e.target && (
                    <span className="ml-2 font-mono text-xs text-muted-foreground">{e.target}</span>
                  )}
                </span>
                <span className="shrink-0 text-muted-foreground">
                  {e.actor} · {timeAgo(e.at)}
                </span>
              </li>
```

- [ ] **Step 3: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors; build OK.

- [ ] **Step 4: Commit**

```bash
git add "app/(admin)/admin/customers/[tenantId]/page.tsx"
git commit -m "feat(audit): friendly labels + target + timeAgo in the admin customer activity (2C)"
```

---

## Deferred acceptance (manual UI — user)

- `/tenant/activity` shows friendly action labels, an actor chip for system/stripe rows, a target column, `timeAgo` timestamps, and Previous/Next pagination; `/tenant/activity?page=2` advances.
- `/admin/customers/[id]` activity section shows friendly labels + target + relative time.

---

## Self-Review

**Spec coverage:**
- Pure label map (all 37) + humanizer + `actionLabel` (spec §A) → Task 1; completeness guard test → Task 1. ✅
- `actorType` field + `getOrgAuditPage` offset reader (spec §B) → Task 2. ✅
- Tenant page: shadcn Table, labels, actor chip, target, timeAgo, `?page` pagination (spec §C) → Task 3. ✅
- Admin section: labels + target + timeAgo (spec §D) → Task 4. ✅
- `metadata` NOT rendered; filter bar absent (spec Out-of-scope) — honored. ✅
- Testing: pure unit tests incl. completeness guard (Task 1) + deferred manual checks (spec §Testing). ✅

**Placeholder scan:** None — every step shows complete code. ✅

**Type consistency:** `actionLabel(action: string)` (Task 1) used in Tasks 3 & 4; the `AuditRow` shape `{ id, action, actorType, actor, target, metadata, at }` is identical in `getOrgAuditLog` (extended) and `getOrgAuditPage` (Task 2) and consumed by the pages; `getOrgAuditPage` returns `{ rows, total, page, pageSize, pageCount }` matching the Task-3 destructure (`{ rows, page, pageCount }`); `timeAgo(iso: string)` reused. ✅

**Note for implementers:** the admin page already imports `timeAgo`; only `actionLabel` is a new import there. The tenant page is a full rewrite (it was a bare `<table>`); confirm no other code imported anything from the old activity page (it exported only the default page component).
