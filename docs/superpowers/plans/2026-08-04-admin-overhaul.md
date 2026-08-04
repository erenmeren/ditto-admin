# Super-Admin Overhaul (Cleanup & Alignment) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove dead invoice/poll-era code from the super-admin panel, fix archived-org leaks in shared queries, and align pin display, health badges, and billing-plan visibility with the tenant side.

**Architecture:** Pure cleanup + alignment on the existing Next.js App Router + Drizzle stack. All data changes stay inside `lib/data.ts` (the single data seam) plus one new tiny pure helper module `lib/archived.ts`. No schema changes, no migrations, no new pages, no nav additions.

**Tech Stack:** Next.js 16, React 19, TypeScript strict, Drizzle ORM (Neon), vitest (`lib/**/*.test.ts`, node env), Tailwind v4 + shadcn (radix-nova).

**Spec:** `docs/superpowers/specs/2026-08-04-admin-overhaul-design.md`

## Global Constraints

- **⚠ `.env.local` points at PRODUCTION Neon.** Tests are pure (no DB). Never run a script that WRITES to the DB during this plan. Read-only queries for verification are OK.
- **No deploy.** Work is committed to a feature branch; the user tests locally and batches the deploy.
- Branch: create `refactor/admin-overhaul` from `main` before Task 1; every task commits to it.
- Gates after every task: `npx tsc --noEmit` and `npx vitest run` must pass. `npm run build` at Tasks 11 and 19 only (it's slow).
- The alert key `"documents-stuck"` in `lib/health.ts:45` is persisted identity — do NOT rename it (its explanatory comment already exists and stays).
- `KpiCard`'s `delta` prop is used with REAL data by the tenant dashboard — do not remove the prop, only the admin overview's fake usage.
- Copy style: sentence case, terse, no exclamation marks — match surrounding strings.

---

### Task 1: Delete dead components

**Files:**
- Delete: `components/placeholder.tsx`
- Delete: `components/fleet-table.tsx`

**Interfaces:** none (both have zero importers).

- [ ] **Step 1: Verify zero importers**

Run: `grep -rn "placeholder\"\|Placeholder\b" app components lib --include="*.tsx" --include="*.ts" | grep -v "components/placeholder" | grep -vi "placeholder=" | grep -vi "placeholder:"`
Expected: no import of `@/components/placeholder`. Repeat for `FleetTable`:
`grep -rn "fleet-table\|FleetTable" app components lib | grep -v "components/fleet-table"`
Expected: no hits.

- [ ] **Step 2: Delete both files**

```bash
git rm components/placeholder.tsx components/fleet-table.tsx
```

- [ ] **Step 3: Gate**

Run: `npx tsc --noEmit && npx vitest run`
Expected: clean compile, all tests pass.

- [ ] **Step 4: Commit**

```bash
git commit -m "chore(admin): delete dead Placeholder and FleetTable components"
```

---

### Task 2: Retire invoice-era audit constants, mark labels legacy

**Files:**
- Modify: `lib/audit.ts:19-25`
- Modify: `lib/audit-labels.ts:10-16`
- Test (existing): `lib/audit-labels.test.ts`

**Interfaces:**
- Produces: `AUDIT` no longer has `subscriptionStatusChanged`, `invoicePaid`, `invoicePaymentFailed`, `invoiceVoid`, `invoiceSent`, `invoiceOverdue`, `billingActivated`. `AUDIT_LABELS` KEEPS all seven strings (legacy display).

- [ ] **Step 1: Verify the constants have no emitters**

Run: `grep -rn "AUDIT\.invoice\|AUDIT\.subscription\|AUDIT\.billingActivated" app lib components --include="*.ts" --include="*.tsx" | grep -v "lib/audit.ts"`
Expected: no hits.

- [ ] **Step 2: Remove the seven constants from `lib/audit.ts`**

Delete these lines from the `AUDIT` object (currently lines 19-25):

```ts
  subscriptionStatusChanged: "subscription.status_changed",
  invoicePaid: "invoice.paid",
  invoicePaymentFailed: "invoice.payment_failed",
  invoiceVoid: "invoice.void",
  invoiceSent: "invoice.sent",
  invoiceOverdue: "invoice.overdue",
  billingActivated: "billing.activated",
```

- [ ] **Step 3: Mark the labels as legacy in `lib/audit-labels.ts`**

Above the `"subscription.status_changed"` entry (line 10), insert:

```ts
  // Legacy display only: the billing pivot (2026-07-04) removed every emitter of
  // the seven actions below, but pre-pivot audit rows still exist in the DB and
  // must keep rendering friendly labels. Do not remove; do not re-emit.
```

- [ ] **Step 4: Gate**

Run: `npx tsc --noEmit && npx vitest run lib/audit-labels.test.ts && npx vitest run`
Expected: all pass. (The "label for every AUDIT constant" test iterates `AUDIT` values — labels are now a superset, which it allows. `actionLabel("invoice.sent")` still returns "Invoice sent" because the label stays.)

- [ ] **Step 5: Commit**

```bash
git add lib/audit.ts lib/audit-labels.ts
git commit -m "chore(audit): retire unemitted invoice/subscription constants; keep labels for legacy rows"
```

---

### Task 3: Remove the dead `money` prop and `formatCurrency`

**Files:**
- Modify: `components/charts.tsx` (prop at ~:51,55 and usages at ~:56-57, ~:134, ~:145)
- Modify: `lib/format.ts:3` (remove `formatCurrency` only)

- [ ] **Step 1: Verify no caller passes `money` and no other `formatCurrency` use**

Run: `grep -rn "money" app components --include="*.tsx" | grep -v charts.tsx` and `grep -rn "formatCurrency" app components lib | grep -v "lib/format.ts" | grep -v charts.tsx`
Expected: no hits for either.

- [ ] **Step 2: Strip the prop**

In `components/charts.tsx`: remove `money` from the tooltip and `BreakdownBarChart` prop types and destructuring; replace the two conditional usages with the plain branch:

```tsx
// tooltip value line becomes:
{formatNumber(Number(p.value))}
// XAxis tickFormatter becomes:
tickFormatter={(v) => formatCompact(Number(v))}
// Tooltip content becomes:
content={<ChartTooltip unit="activations" />}
```

Remove the `formatCurrency` import from charts.tsx and delete the `formatCurrency` function from `lib/format.ts` (keep `formatNumber` / `formatCompact` — widely used).

- [ ] **Step 3: Gate + commit**

Run: `npx tsc --noEmit && npx vitest run`

```bash
git add components/charts.tsx lib/format.ts
git commit -m "chore(charts): drop invoice-era money prop and formatCurrency"
```

---

### Task 4: Remove the unreachable `trial` tenant status

**Files:**
- Modify: `components/tenant-status-badge.tsx:10-14` (delete the `trial` entry from `META`)
- Modify: `lib/types.ts:9` (remove `"trial"` from the `TenantStatus` union)

- [ ] **Step 1: Verify unreachable**

Run: `grep -rn "\"trial\"\|'trial'" app components lib --include="*.ts" --include="*.tsx"`
Expected: only the two sites above (`mapTenantStatus` in `lib/data.ts:245-248` only emits `active`/`suspended`).

- [ ] **Step 2: Delete both, gate, commit**

Run: `npx tsc --noEmit && npx vitest run`

```bash
git add components/tenant-status-badge.tsx lib/types.ts
git commit -m "chore(admin): remove unreachable trial tenant status"
```

---

### Task 5: Un-export `getTenantSummaries`, drop unused `AdminOverview.daily`

**Files:**
- Modify: `lib/data.ts:1093` (remove `export` keyword), `lib/data.ts:1140-1155` (remove `daily` from the interface, the `sumSeries(bundles.map((b) => dailySeries(b)))` computation, and the `daily` key in the return)

**Interfaces:**
- Produces: `AdminOverview` = `{ activationsThisMonth, activeDevices, totalDevices, totalCustomers, totalStores, monthly, topCustomers }` (no `daily`). `getTenantSummaries` stays as an internal function (Task 12 modifies it).

- [ ] **Step 1: Verify** — `grep -rn "getTenantSummaries" app components lib | grep -v "lib/data.ts"` → no hits; `grep -rn "\.daily" app components --include="*.tsx" | grep -v tenant` → no admin usage (`dailySeries` itself stays — the tenant dashboard uses it).
- [ ] **Step 2: Apply the three edits.**
- [ ] **Step 3: Gate + commit**

```bash
git add lib/data.ts
git commit -m "chore(data): unexport getTenantSummaries; stop computing unused admin daily series"
```

---

### Task 6: Make `ExportButton` data required

**Files:**
- Modify: `components/export-button.tsx` (~:31-36)

- [ ] **Step 1:** Make `headers` and `rows` required in the prop type (drop the `?`/`| undefined`), and delete the fallback block:

```tsx
    if (!headers || !rows) {
      toast.info("Export not available", {
        description: "There's nothing to export here yet.",
      });
      return;
    }
```

If the `toast` import becomes unused, remove it.
- [ ] **Step 2: Gate + commit** (the only caller, `app/(admin)/admin/billing/page.tsx:41-46`, always passes both).

```bash
git add components/export-button.tsx
git commit -m "chore(admin): ExportButton data is required — remove unreachable empty-export path"
```

---

### Task 7: Copy & terminology sweep

**Files:**
- Modify: `components/devices/command-bar.tsx:36`
- Modify: `app/(admin)/admin/page.tsx` (:2 import, :43 delta, :45 icon)
- Modify: `app/(admin)/admin/customers/[tenantId]/page.tsx` (FileText import + usage at ~:160)
- Modify: `app/(admin)/admin/devices/[deviceId]/page.tsx` (:3 import, :87-88 icons)
- Modify: `lib/nav.ts:30`
- Modify: `components/admin/integration-health-card.tsx:86`
- Modify: `lib/data.ts:8` and `lib/data.ts:~1640` (comments)
- Modify: `components/app-shell.tsx:130` (comment)

- [ ] **Step 1: command-bar success copy** — replace line 36:

```tsx
      setMsg(r.ok ? `${type} sent — pushed to the device over MQTT.` : r.error);
```

- [ ] **Step 2: overview page** — in `app/(admin)/admin/page.tsx` change the import `FileText` → `Zap` (line 2) and the first KpiCard:

```tsx
        <KpiCard
          label="Activations this month"
          value={formatCompact(o.activationsThisMonth)}
          hint="platform-wide"
          icon={Zap}
        />
```

(the `delta={12.1}` line is deleted — it was a hardcoded fake trend).

- [ ] **Step 3: customer detail + device detail icons** — same `FileText` → `Zap` swap in both files (`[tenantId]/page.tsx` one usage, `[deviceId]/page.tsx` two usages). Keep every other icon as is.
- [ ] **Step 4: nav label** — `lib/nav.ts:30`: `{ label: "Billing & Credits", href: "/admin/billing", icon: Wallet },`
- [ ] **Step 5: Stripe live copy** — `integration-health-card.tsx:86`: `"Charges run against the live Stripe account."`
- [ ] **Step 6: comments** — `lib/data.ts:8`: replace `(invoice amount)` with `(credit pack pricing)`. `lib/data.ts:~1640` section comment: replace with `// ---- Tenant billing data (credit balance, packs, plan) ----`. `components/app-shell.tsx:130`: replace the `printed slip` phrase with `brand accent`.
- [ ] **Step 7: Gate**

Run: `npx tsc --noEmit && npx vitest run && grep -rn "next check-in\|delta={12.1}\|Billing & Revenue" app components lib`
Expected: green tests, grep finds nothing.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "fix(admin): purge poll/invoice-era copy — MQTT command wording, real-only badges, nav label"
```

---

### Task 8: `lib/archived.ts` pure helper (TDD)

**Files:**
- Create: `lib/archived.ts`
- Test: `lib/archived.test.ts`

**Interfaces:**
- Produces: `excludeArchived<T extends { archivedAt: Date | null }>(rows: T[]): T[]` — used by Tasks 9, 10, 16.

- [ ] **Step 1: Write the failing test**

```ts
// lib/archived.test.ts
import { describe, expect, it } from "vitest";
import { excludeArchived } from "./archived";

describe("excludeArchived", () => {
  it("drops rows with archivedAt set, keeps null", () => {
    const rows = [
      { id: "a", archivedAt: null },
      { id: "b", archivedAt: new Date("2026-07-10") },
      { id: "c", archivedAt: null },
    ];
    expect(excludeArchived(rows).map((r) => r.id)).toEqual(["a", "c"]);
  });

  it("returns an empty array untouched", () => {
    expect(excludeArchived([])).toEqual([]);
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npx vitest run lib/archived.test.ts` → FAIL (module not found).
- [ ] **Step 3: Implement**

```ts
// lib/archived.ts
// Pure: archive-lifecycle row filter. Admin queries left-join tenant_settings
// and pass rows through this so "archived org" means the same thing everywhere
// (a missing settings row counts as not archived, matching loadAllOrgs).
export function excludeArchived<T extends { archivedAt: Date | null }>(rows: T[]): T[] {
  return rows.filter((r) => r.archivedAt === null);
}
```

- [ ] **Step 4: Run to verify it passes** — `npx vitest run lib/archived.test.ts` → PASS.
- [ ] **Step 5: Commit**

```bash
git add lib/archived.ts lib/archived.test.ts
git commit -m "feat(data): excludeArchived helper — one definition of archived across admin queries"
```

---

### Task 9: Archive + claimed filters in `getPlatformHealth` and `getAlertInputs`

**Files:**
- Modify: `lib/data.ts` — `getPlatformHealth` (:1805-1930) and `getAlertInputs` (:1937-1982)

**Interfaces:**
- Consumes: `excludeArchived` from Task 8; `settingsTable` is already imported in data.ts (used by `loadAllOrgs`). Ensure `isNull` is in the drizzle-orm import list (add it if missing).
- Produces: unchanged return types; fleet numbers now = claimed devices in non-archived orgs; `inactiveTenants` excludes archived orgs. `getAlertInputs` feeds `lib/alerts-sync.ts:86` — phantom alerts stop being produced.

- [ ] **Step 1: Fleet counts** — in `getPlatformHealth`, replace the `devRows` query:

```ts
    const devRows = excludeArchived(
      await db
        .select({
          status: deviceTable.status,
          lastSeenAt: deviceTable.lastSeenAt,
          archivedAt: settingsTable.archivedAt,
        })
        .from(deviceTable)
        .leftJoin(settingsTable, eq(settingsTable.organizationId, deviceTable.organizationId))
        .where(isNotNull(deviceTable.claimedAt)),
    );
```

- [ ] **Step 2: Stale predicate** — extend `stalePred` with `isNotNull(deviceTable.claimedAt)` and `isNull(settingsTable.archivedAt)`, and add `.leftJoin(settingsTable, eq(settingsTable.organizationId, deviceTable.organizationId))` to BOTH the `staleRows` query and the `staleCount` query (the predicate references the joined table, so the join is mandatory in each).
- [ ] **Step 3: Inactive tenants (both functions)** — replace the `allOrgs` select in `getPlatformHealth` (:1876) and `getAlertInputs` (:1962) with:

```ts
    const allOrgs = excludeArchived(
      await db
        .select({ id: orgTable.id, name: orgTable.name, archivedAt: settingsTable.archivedAt })
        .from(orgTable)
        .leftJoin(settingsTable, eq(settingsTable.organizationId, orgTable.id)),
    );
```

(downstream `.filter/.map` code is shape-compatible — `archivedAt` rides along harmlessly).
- [ ] **Step 4: Same claimed+archived join/predicate on `getAlertInputs`'s `staleCount` query** (mirror Step 2 — the comment above the function says it must mirror `getPlatformHealth`).
- [ ] **Step 5: Gate** — `npx tsc --noEmit && npx vitest run`.
- [ ] **Step 6: Commit**

```bash
git add lib/data.ts
git commit -m "fix(health): fleet counts and inactive-tenant alerts exclude archived orgs and unclaimed devices"
```

---

### Task 10: Archive filters in the two credit queries + billing footnote

**Files:**
- Modify: `lib/data.ts` — `getCreditUsageAllOrgs` (:2166-2179), `getCreditsOverview` (:2214-2250)
- Modify: `app/(admin)/admin/billing/page.tsx` (footnote)

**Interfaces:**
- Produces: same return types; archived orgs absent from both.

- [ ] **Step 1: `getCreditUsageAllOrgs`** — add the settings join + predicate:

```ts
    .leftJoin(orgTable, eq(orgTable.id, creditLedgerTable.organizationId))
    .leftJoin(settingsTable, eq(settingsTable.organizationId, creditLedgerTable.organizationId))
    .where(
      and(
        inArray(creditLedgerTable.kind, ["settle", "spend"]),
        gte(creditLedgerTable.createdAt, since),
        isNull(settingsTable.archivedAt),
      ),
    )
```

- [ ] **Step 2: `getCreditsOverview`** — fetch `archivedAt` with the orgs, filter, and scope the other two row sets to surviving org ids:

```ts
  const [orgRows, ledgerRows, balanceRows] = await Promise.all([
    db
      .select({ id: orgTable.id, name: orgTable.name, archivedAt: settingsTable.archivedAt })
      .from(orgTable)
      .leftJoin(settingsTable, eq(settingsTable.organizationId, orgTable.id)),
    /* ledger + balance selects unchanged */
  ]);
  const orgs = excludeArchived(orgRows);
  const activeIds = new Set(orgs.map((o) => o.id));
  const nameOf = new Map(orgs.map((o) => [o.id, o.name]));
  // then filter before mapping into rollupCredits:
  ledgerRows.filter((r) => activeIds.has(r.organizationId)).map(...)
  balanceRows.filter((b) => activeIds.has(b.organizationId)).map(...)
```

- [ ] **Step 3: Footnote** — in `app/(admin)/admin/billing/page.tsx`, under the KPI grid add:

```tsx
      <p className="text-xs text-muted-foreground">
        Archived customers&apos; frozen credits are excluded from these totals.
      </p>
```

- [ ] **Step 4: Gate + commit**

```bash
git add lib/data.ts "app/(admin)/admin/billing/page.tsx"
git commit -m "fix(billing): archived orgs leave credit rollups; frozen credits no longer counted as liability"
```

---

### Task 11: Phantom-alert verification (READ-ONLY — prod DB)

**Files:** none modified. `.env.local` is PRODUCTION — read-only queries only, do NOT run `syncAlerts` or any write.

- [ ] **Step 1: Confirm inputs are clean** — run a scratch read (from repo root):

```bash
npx tsx -e "
import './lib/db/load-env';
const { getAlertInputs } = await import('./lib/data');
const i = await getAlertInputs();
console.log('inactiveTenants:', i.inactiveTenants.map(t => t.name));
"
```

Expected: no archived customer names in the list (compare against `/admin/customers` Archived tab).
- [ ] **Step 2: Check open phantom alerts** — read the `alert` table:

```bash
npx tsx -e "
import './lib/db/load-env';
const { db } = await import('./lib/db');
const { alert } = await import('./lib/db/schema');
const { isNull } = await import('drizzle-orm');
const rows = await db.select({ key: alert.key, msg: alert.message }).from(alert).where(isNull(alert.resolvedAt));
console.log(rows);
"
```

Record any `tenant-inactive:<id>` rows belonging to archived orgs in the task notes. **Do not resolve them locally** — the deployed cron's own resolve pass closes alerts whose input disappears; verify that after the user deploys. If the cron's resolve logic turns out NOT to close missing-input alerts (check `lib/alerts-sync.ts` while here and note the finding), add a follow-up task for a one-time cleanup script to run post-deploy.
- [ ] **Step 3: Build gate** — `npm run build` → passes.
- [ ] **Step 4: Commit nothing** (verification only). Note findings in the PR/handoff message.

---

### Task 12: Health badge unification (list = detail inputs)

**Files:**
- Modify: `lib/data.ts` — `summarize` (:322-360), `getTenantSummaries` (:1093-1098), `getAdminOverview` (:1151-1153)
- Modify: `lib/tenant-health.ts:12-13` (comments)
- Test: `lib/tenant-health.test.ts` (new)

**Interfaces:**
- Produces: `function summarize(b: OrgBundle, extras?: { stuckPendingCount?: number; lastActivityAt?: Date | null }): TenantSummary`. `getTenantSummaries` gathers extras with two grouped queries. `getAdminOverview` keeps calling `summarize(b)` without extras — its `TenantSummary.health` is not rendered anywhere on the overview (only status badges are); note this with a one-line comment at the call site.

- [ ] **Step 1: Write the contract tests** (the pure function already exists — these pin the behavior the list path now relies on)

```ts
// lib/tenant-health.test.ts
import { describe, expect, it } from "vitest";
import { tenantHealthLevel } from "./tenant-health";
import { INACTIVE_DAYS } from "./health";

const now = new Date("2026-08-04T12:00:00Z");

describe("tenantHealthLevel — unified list/detail inputs", () => {
  it("warns on stuck pending even with the whole fleet online", () => {
    expect(
      tenantHealthLevel(
        { deviceCount: 3, onlineCount: 3, offlineCount: 0, stuckPendingCount: 1, lastActivityAt: now },
        now,
      ),
    ).toBe("warning");
  });

  it("warns after INACTIVE_DAYS of no activity", () => {
    const stale = new Date(now.getTime() - (INACTIVE_DAYS + 1) * 86_400_000);
    expect(
      tenantHealthLevel(
        { deviceCount: 1, onlineCount: 1, offlineCount: 0, stuckPendingCount: 0, lastActivityAt: stale },
        now,
      ),
    ).toBe("warning");
  });

  it("stays healthy for a never-active org with everything online", () => {
    expect(
      tenantHealthLevel(
        { deviceCount: 1, onlineCount: 1, offlineCount: 0, stuckPendingCount: 0, lastActivityAt: null },
        now,
      ),
    ).toBe("healthy");
  });
});
```

- [ ] **Step 2: Run them** — `npx vitest run lib/tenant-health.test.ts` → all three PASS (if any fails, STOP: the pure function's contract differs from the spec's assumption — investigate before threading the inputs).
- [ ] **Step 3: Thread extras through `summarize`** — change the signature and the `tenantHealthLevel` call:

```ts
function summarize(
  b: OrgBundle,
  extras?: { stuckPendingCount?: number; lastActivityAt?: Date | null },
): TenantSummary {
  // ...unchanged until the health call:
  const health = tenantHealthLevel(
    {
      deviceCount: allDevices.length,
      onlineCount,
      offlineCount,
      stuckPendingCount: extras?.stuckPendingCount ?? 0,
      lastActivityAt: extras?.lastActivityAt ?? null,
    },
    now,
  );
```

- [ ] **Step 4: Gather extras in `getTenantSummaries`** (two grouped queries, no N+1):

```ts
async function getTenantSummaries(opts?: { includeArchived?: boolean }): Promise<TenantSummary[]> {
  const bundles = await loadAllOrgs(opts);
  const stuckCutoff = new Date(Date.now() - STUCK_PENDING_MINUTES * 60_000);
  const [stuckRows, lastRows] = await Promise.all([
    db
      .select({ org: deviceCommand.organizationId, c: count() })
      .from(deviceCommand)
      .where(and(eq(deviceCommand.type, "trigger"), eq(deviceCommand.status, "pending"), lt(deviceCommand.createdAt, stuckCutoff)))
      .groupBy(deviceCommand.organizationId),
    db
      .select({ org: deviceCommand.organizationId, last: max(deviceCommand.createdAt) })
      .from(deviceCommand)
      .where(and(eq(deviceCommand.type, "trigger"), eq(deviceCommand.status, "acked")))
      .groupBy(deviceCommand.organizationId),
  ]);
  const stuckBy = new Map(stuckRows.map((r) => [r.org, Number(r.c)]));
  const lastBy = new Map<string, Date>();
  for (const r of lastRows) if (r.last) lastBy.set(r.org, r.last);
  return bundles.map((b) =>
    summarize(b, { stuckPendingCount: stuckBy.get(b.org.id) ?? 0, lastActivityAt: lastBy.get(b.org.id) ?? null }),
  );
}
```

(`STUCK_PENDING_MINUTES` is already imported — `getCustomerDetail` uses it.)
- [ ] **Step 5: Comments** — `lib/tenant-health.ts:12-13`: change both trailing comments to `// supplied by both the list and detail paths`. In `getAdminOverview`, above `bundles.map(summarize)` add: `// no extras: the overview renders status badges, never TenantSummary.health`. Wrap as `bundles.map((b) => summarize(b))` so the signature change compiles.
- [ ] **Step 6: Gate** — `npx tsc --noEmit && npx vitest run`.
- [ ] **Step 7: Commit**

```bash
git add lib/data.ts lib/tenant-health.ts lib/tenant-health.test.ts
git commit -m "fix(admin): customer-list health badge uses the same inputs as the detail page"
```

---

### Task 13: Effective pinned QR on the admin device detail

**Files:**
- Modify: `app/(admin)/admin/devices/[deviceId]/page.tsx` (fetch + the `Pinned QR` specs row at ~:72)

**Interfaces:**
- Consumes: `getDevicePinContext(organizationId, deviceId): Promise<DevicePinContext | null>` from `lib/data.ts:977` (`{ pinMode, inheritedUrl, inheritedSource }`); the page's `device` view-model already carries `tenantId` and `pinnedUrl`.

- [ ] **Step 1: Fetch pin context** — alongside the existing queries in the page component:

```tsx
  const pinCtx = await getDevicePinContext(device.tenantId, device.id);
```

Add `getDevicePinContext` to the existing `@/lib/data` import.
- [ ] **Step 2: Replace the specs row value**

```tsx
  const pinValue = !pinCtx
    ? "—"
    : pinCtx.pinMode === "custom"
      ? `${device.pinnedUrl ?? "—"} (device)`
      : pinCtx.pinMode === "none"
        ? "Disabled"
        : pinCtx.inheritedUrl
          ? `${pinCtx.inheritedUrl} (${pinCtx.inheritedSource})`
          : "—";
```

and use it: `{ icon: Pin, label: "Pinned QR", value: pinValue, mono: true },`
- [ ] **Step 3: Verify pure resolution coverage exists** — `npx vitest run lib/pin-resolve.test.ts` (if that file doesn't exist, run `npx vitest run` and confirm pin tests live elsewhere; the resolution logic itself is pre-tested — this task only wires it).
- [ ] **Step 4: Gate + commit**

```bash
git add "app/(admin)/admin/devices/[deviceId]/page.tsx"
git commit -m "fix(admin): device detail shows the effective pinned QR with its inherit source"
```

---

### Task 14: Billing-plan badge on the customers list

**Files:**
- Modify: `lib/data.ts` — `summarize` return (+ `TenantSummary` type where it's declared — find with `grep -n "interface TenantSummary" lib/*.ts`; it carries `archivedAt`, so extend the same declaration)
- Create: `components/billing/plan-badge.tsx`
- Modify: `app/(admin)/admin/customers/page.tsx` (new column)

**Interfaces:**
- Produces: `TenantSummary.billingPlan: "credits" | "flat" | "base_usage"` and `<PlanBadge plan={...} />` (also used by Task 16).

- [ ] **Step 1: Data** — in `summarize`'s return object add:

```ts
    billingPlan: b.settings?.billingPlan ?? "credits",
```

and add the field to the `TenantSummary` interface with the union type above (reuse the existing `BillingPlan` type if one is exported from `lib/types.ts` — check with `grep -n "BillingPlan" lib/types.ts lib/data.ts`).
- [ ] **Step 2: Badge component**

```tsx
// components/billing/plan-badge.tsx
const LABELS: Record<string, string> = {
  credits: "Credits",
  flat: "Flat",
  base_usage: "Base + Usage",
};

export function PlanBadge({ plan }: { plan: string }) {
  return (
    <span className="inline-flex items-center rounded-full bg-muted px-2 py-0.5 text-xs font-medium text-muted-foreground">
      {LABELS[plan] ?? plan}
    </span>
  );
}
```

- [ ] **Step 3: Column** — in `app/(admin)/admin/customers/page.tsx` add `<TableHead className="text-center">Plan</TableHead>` after the `Health` head (line ~103) and the matching cell in the row render:

```tsx
<TableCell className="text-center"><PlanBadge plan={c.billingPlan} /></TableCell>
```

- [ ] **Step 4: Gate + commit**

```bash
git add lib/data.ts components/billing/plan-badge.tsx "app/(admin)/admin/customers/page.tsx"
git commit -m "feat(admin): billing-plan badge on the customers list"
```

---

### Task 15: Plan-aware credits note on the customer detail

**Files:**
- Modify: `app/(admin)/admin/customers/[tenantId]/page.tsx` (Credits card, header area ~:230-246)

- [ ] **Step 1:** Inside the Credits card content, before the balance display, add (flat only — on `base_usage` credits still pay for overage, so no note):

```tsx
          {tenant.billingPlan === "flat" && (
            <p className="text-sm text-muted-foreground">
              This tenant is on the flat plan — triggers do not consume credits.
            </p>
          )}
```

- [ ] **Step 2: Gate + commit**

```bash
git add "app/(admin)/admin/customers/[tenantId]/page.tsx"
git commit -m "feat(admin): flat-plan note on the customer credits card"
```

---

### Task 16: Plan mix on the billing page

**Files:**
- Modify: `lib/data.ts` — new `getPlanMix()` next to `getCreditsOverview`, and extend `getCreditsOverview`'s org select with `billingPlan`
- Modify: `app/(admin)/admin/billing/page.tsx` (plan-mix card + plan column)

**Interfaces:**
- Produces: `getPlanMix(): Promise<{ credits: number; flat: number; baseUsage: number; flatDevices: number; baseUsageDevices: number }>` and `getCreditsOverview()` return gains `planByOrg: Record<string, string>`.

- [ ] **Step 1: `getPlanMix`**

```ts
/** Active-tenant count per billing plan + claimed-device counts for the
 *  subscription tracks (the revenue proxy while Stripe figures stay out of scope). */
export async function getPlanMix(): Promise<{
  credits: number; flat: number; baseUsage: number; flatDevices: number; baseUsageDevices: number;
}> {
  const orgs = excludeArchived(
    await db
      .select({ id: orgTable.id, archivedAt: settingsTable.archivedAt, plan: settingsTable.billingPlan })
      .from(orgTable)
      .leftJoin(settingsTable, eq(settingsTable.organizationId, orgTable.id)),
  );
  const planOf = new Map(orgs.map((o) => [o.id, o.plan ?? "credits"]));
  const counts = { credits: 0, flat: 0, base_usage: 0 } as Record<string, number>;
  for (const p of planOf.values()) counts[p] = (counts[p] ?? 0) + 1;

  const devRows = await db
    .select({ org: deviceTable.organizationId, c: count() })
    .from(deviceTable)
    .where(isNotNull(deviceTable.claimedAt))
    .groupBy(deviceTable.organizationId);
  let flatDevices = 0, baseUsageDevices = 0;
  for (const r of devRows) {
    const p = planOf.get(r.org);
    if (p === "flat") flatDevices += Number(r.c);
    else if (p === "base_usage") baseUsageDevices += Number(r.c);
  }
  return { credits: counts.credits, flat: counts.flat, baseUsage: counts.base_usage, flatDevices, baseUsageDevices };
}
```

- [ ] **Step 2: `planByOrg`** — in `getCreditsOverview`, add `plan: settingsTable.billingPlan` to the org select (the settings join exists after Task 10); build `const planByOrg = Object.fromEntries(orgs.map((o) => [o.id, o.plan ?? "credits"]));` and include it in the return: `return { ...rollupCredits(...), planByOrg };` (adjust the function's return type accordingly).
- [ ] **Step 3: Page** — in `app/(admin)/admin/billing/page.tsx`: fetch both (`Promise.all([getCreditsOverview(), getPlanMix()])`), add a Plan-mix card between the KPI grid and the per-tenant table:

```tsx
      <Card>
        <CardHeader>
          <CardTitle>Plan mix</CardTitle>
          <CardDescription>Active tenants per billing plan</CardDescription>
        </CardHeader>
        <CardContent className="grid gap-4 sm:grid-cols-3">
          <div>
            <p className="text-2xl font-semibold tabular-nums">{mix.credits}</p>
            <p className="text-sm text-muted-foreground">Credits</p>
          </div>
          <div>
            <p className="text-2xl font-semibold tabular-nums">{mix.flat}</p>
            <p className="text-sm text-muted-foreground">Flat · {formatNumber(mix.flatDevices)} devices</p>
          </div>
          <div>
            <p className="text-2xl font-semibold tabular-nums">{mix.baseUsage}</p>
            <p className="text-sm text-muted-foreground">Base + usage · {formatNumber(mix.baseUsageDevices)} devices</p>
          </div>
        </CardContent>
      </Card>
```

and a `Plan` column in the per-tenant table using `<PlanBadge plan={credits.planByOrg[t.orgId] ?? "credits"} />` (import from Task 14). Add `Plan` to `exportHeaders`/`exportRows` too.
- [ ] **Step 4: Gate + commit**

```bash
git add lib/data.ts "app/(admin)/admin/billing/page.tsx"
git commit -m "feat(admin): plan mix and per-tenant plan on the billing page"
```

---

### Task 17: Firmware page query moves into the data layer

**Files:**
- Modify: `lib/data.ts` (new function near the other admin functions)
- Modify: `app/(admin)/admin/firmware/page.tsx:1-15`

**Interfaces:**
- Produces: `getFirmwareReleases(limit = 50)` returning the full `firmwareRelease` rows ordered newest-first.

- [ ] **Step 1: Data function**

```ts
/** Newest-first firmware releases for the admin Firmware page. */
export async function getFirmwareReleases(limit = 50) {
  return db.select().from(firmwareRelease).orderBy(desc(firmwareRelease.createdAt)).limit(limit);
}
```

(`firmwareRelease` needs adding to the schema import in data.ts if absent; `desc` is already imported.)
- [ ] **Step 2: Page** — replace the inline query with `const releases = await getFirmwareReleases();` and drop the now-unused `db` / `firmwareRelease` / `desc` imports from the page.
- [ ] **Step 3: Gate + commit**

```bash
git add lib/data.ts "app/(admin)/admin/firmware/page.tsx"
git commit -m "refactor(admin): firmware page reads releases through lib/data"
```

---

### Task 18: Turkish super-admin manual fixes

**Files:**
- Modify: `docs/manuals/tr/super-admin-kilavuzu.md` (targeted lines; no rewrite)

- [ ] **Step 1: Apply the six fixes** (line numbers approximate — locate by the quoted text):
  1. `:80` — replace the polling sentence (`"düzenli olarak yoklar (polling)"` and surrounding claim) with: cihaz komutları MQTT üzerinden anında alır; kalıcı bir MQTT bağlantısı vardır, yoklama yoktur.
  2. `:695` and `:723` — same poll→MQTT-push correction where remote commands are described ("cihaz bir sonraki yoklamada alır" → "komut MQTT ile anında iletilir").
  3. `:224` — `"printers online"` quote → `"screens online"`.
  4. `:638-639` — delete-dialog quote → `"...and its command history"` / aktivasyon geçmişi ifadesi (match `components/device-row-actions.tsx:252` and `components/unclaimed-device-delete.tsx:66-67`).
  5. `:174` — `"Faturalandırma ve Gelir (Billing & Revenue)"` → `"Faturalandırma ve Krediler (Billing & Credits)"`.
  6. `:504-510` — provision-dialog quotes `"Provision a new printer"` / `"e.g. Printer 1"` → `"Provision a new screen"` / `"e.g. Screen 1"`.
- [ ] **Step 2: Sweep check** — `grep -n "polling\|yoklar\|printer\|Printer\|Billing & Revenue\|document history" docs/manuals/tr/super-admin-kilavuzu.md` → remaining hits are only intentional ones (e.g. the manual's own `documents-stuck` historical-artifact note).
- [ ] **Step 3: Commit**

```bash
git add docs/manuals/tr/super-admin-kilavuzu.md
git commit -m "docs(tr): super-admin manual catches up with MQTT push, screen naming, credits nav"
```

---

### Task 19: Final gate

- [ ] **Step 1:** `npx tsc --noEmit` → clean.
- [ ] **Step 2:** `npx vitest run` → full suite green (expect 324+ pre-existing + new archived/tenant-health tests).
- [ ] **Step 3:** `npm run build` → passes.
- [ ] **Step 4:** Residue sweep:

```bash
grep -rn "next check-in\|Billing & Revenue\|delta={12.1}\|FileText" "app/(admin)" components/devices/command-bar.tsx lib/nav.ts
```

Expected: zero hits (`FileText` may legitimately remain in tenant-side files — the grep is scoped to admin surfaces).
- [ ] **Step 5:** Leave the branch unpushed/undeployed; hand off with: task-by-task commit list, the Task 11 phantom-alert findings, and the reminder that alert cleanup is verified after the user's batched deploy.
