# Repoint Operational Metrics to Trigger Activity — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the tenant dashboards, per-store analytics, and platform health rollups reflect live device-trigger activity instead of the now-frozen `document` table.

**Architecture:** All metric SQL lives in `lib/data.ts` (the data seam), which feeds pure IO-free helpers in `lib/analytics.ts` / `lib/eco.ts` / `lib/health.ts`. We (1) swap the SQL source from `document` to acked trigger commands, (2) remap the health card to the trigger status model, then (3) rename the "documents" vocabulary to "activations" across the metric path. Billing is deliberately untouched.

**Tech Stack:** Next.js 16 App Router, Drizzle ORM over Neon HTTP, Vitest, TypeScript strict.

## Global Constraints

- **Metric unit:** one event = one **acked trigger** = a `device_command` row where `type = 'trigger' AND status = 'acked'`, bucketed by `created_at`. Copy these filters verbatim into every repointed query.
- **Store attribution:** `device_command` has no `store_id`. Join `device_command → device` on `device_command.deviceId = device.id`, then scope/group by `device.storeId`. (`deviceCommand` and `deviceTable` are already imported in `lib/data.ts`.)
- **Org attribution:** use `device_command.organizationId` directly (non-null; no join).
- **`.env.local` points at PRODUCTION Neon.** Every verification step MUST be **read-only** — never `INSERT`, `db:seed`, or write during verification. Two real acked triggers already exist in prod to verify against.
- **Billing is OUT OF SCOPE — do NOT modify:** `lib/billing-engine.ts`, `getApiUsage` (`/v1/usage`), `invoice.documentCount`, the `Invoice.documents` type field, `app/(tenant)/tenant/billing/*`, `app/(admin)/admin/billing/*` *money* logic, `lib/billing/usage-metering.ts`. (The admin billing page reads the metric field `t.documentsThisMonth` for an activity column — that field rename in Task 3 is allowed; its invoice money is not.)
- **Term:** user-facing metric noun is **"Activations"** (exception: eco copy — see Task 3).
- **Every task ends green:** `npm run build` (Next typecheck+compile) AND `npm test` (Vitest) both pass before commit.
- **Verification scripts** go in the scratchpad dir, are run with `npx tsx`, import `./lib/db/load-env.ts` FIRST, use `sql.query(\`...\`)` (tagged-template `sql\`\`` rejects string calls), and are deleted after use. They are NOT committed.

---

### Task 1: Repoint tenant + store metric SQL source (document → acked triggers)

Swap the query bodies in `loadOrg`, `getStoreAnalytics`, and `getStoresAnalytics` from `documentTable` to acked trigger commands. **No field or label renames** — the view-model shape is identical, so the whole app still builds and every UI stays green. This is the behavior change that makes dashboards truthful.

**Files:**
- Modify: `lib/data.ts` — `loadOrg` (~lines 115–163), `getStoreAnalytics` (~lines 467–481), `getStoresAnalytics` (~lines 518–524)
- Verify (read-only, scratchpad): `<scratchpad>/verify-task1.mjs`

**Interfaces:**
- Consumes: `deviceCommand`, `deviceTable` (already imported), Drizzle `sql/and/eq/gte/count`.
- Produces: no signature changes. `loadOrg` still returns `{ deviceCountRows-derived todayByDevice/monthByDevice, dailyBuckets, monthlyBuckets, ... }`; `getStoreAnalytics`/`getStoresAnalytics` unchanged return types.

- [ ] **Step 1: In `loadOrg`, repoint the day/month expressions and org-scope predicate.** Replace (lines ~115–121):

```ts
  const dayExpr = sql<string>`to_char(date_trunc('day', ${documentTable.createdAt}), 'YYYY-MM-DD')`;
  const monthExpr = sql<string>`to_char(date_trunc('month', ${documentTable.createdAt}), 'YYYY-MM')`;
  const orgScoped = (sinceStr: string) =>
    and(
      eq(documentTable.organizationId, organizationId),
      sql`${documentTable.createdAt} >= ${sinceStr}::timestamp`,
    );
```

with:

```ts
  const dayExpr = sql<string>`to_char(date_trunc('day', ${deviceCommand.createdAt}), 'YYYY-MM-DD')`;
  const monthExpr = sql<string>`to_char(date_trunc('month', ${deviceCommand.createdAt}), 'YYYY-MM')`;
  // Metric = acked trigger commands (a QR the device actually rendered).
  const orgScoped = (sinceStr: string) =>
    and(
      eq(deviceCommand.organizationId, organizationId),
      eq(deviceCommand.type, "trigger"),
      eq(deviceCommand.status, "acked"),
      sql`${deviceCommand.createdAt} >= ${sinceStr}::timestamp`,
    );
```

- [ ] **Step 2: In `loadOrg`, repoint the three grouped queries.** Replace the `deviceCountRows`, `dailyBuckets`, `monthlyBuckets` selects (lines ~143–163) with:

```ts
    db
      .select({
        deviceId: deviceCommand.deviceId,
        today: sql<number>`count(*) FILTER (WHERE ${deviceCommand.createdAt} >= ${todayStartStr}::timestamp)`.mapWith(
          Number,
        ),
        month: sql<number>`count(*)`.mapWith(Number),
      })
      .from(deviceCommand)
      .where(orgScoped(monthStartStr))
      .groupBy(deviceCommand.deviceId),
    db
      .select({ bucket: dayExpr, count: count() })
      .from(deviceCommand)
      .where(orgScoped(since30Str))
      .groupBy(dayExpr),
    db
      .select({ bucket: monthExpr, count: count() })
      .from(deviceCommand)
      .where(orgScoped(since9moStr))
      .groupBy(monthExpr),
```

(The `deviceCountRows` consumer loop below keeps its `if (!r.deviceId) continue;` guard — harmless since `device_command.deviceId` is non-null.)

- [ ] **Step 3: In `getStoreAnalytics`, repoint to a device-joined trigger scope.** Replace lines ~467–481:

```ts
  const dayExpr = sql<string>`to_char(date_trunc('day', ${documentTable.createdAt}), 'YYYY-MM-DD')`;
  const monthExpr = sql<string>`to_char(date_trunc('month', ${documentTable.createdAt}), 'YYYY-MM')`;
  const localTs = sql`((${documentTable.createdAt} AT TIME ZONE 'UTC') AT TIME ZONE ${store.timezone})`;
  const dowExpr = sql<number>`extract(dow from ${localTs})::int`;
  const hourExpr = sql<number>`extract(hour from ${localTs})::int`;
  const scoped = (since: Date) =>
    and(eq(documentTable.storeId, storeId), gte(documentTable.createdAt, since));

  const [dailyRows, monthlyRows, gridRows] = await Promise.all([
    db.select({ bucket: dayExpr, count: count() }).from(documentTable).where(scoped(since30)).groupBy(dayExpr),
    db.select({ bucket: monthExpr, count: count() }).from(documentTable).where(scoped(since9mo)).groupBy(monthExpr),
    db.select({ dow: dowExpr, hour: hourExpr, count: count() }).from(documentTable).where(scoped(since90)).groupBy(sql`1`, sql`2`),
  ]);
```

with (join `device_command → device`, scope by `device.storeId`):

```ts
  const dayExpr = sql<string>`to_char(date_trunc('day', ${deviceCommand.createdAt}), 'YYYY-MM-DD')`;
  const monthExpr = sql<string>`to_char(date_trunc('month', ${deviceCommand.createdAt}), 'YYYY-MM')`;
  const localTs = sql`((${deviceCommand.createdAt} AT TIME ZONE 'UTC') AT TIME ZONE ${store.timezone})`;
  const dowExpr = sql<number>`extract(dow from ${localTs})::int`;
  const hourExpr = sql<number>`extract(hour from ${localTs})::int`;
  const scoped = (since: Date) =>
    and(
      eq(deviceTable.storeId, storeId),
      eq(deviceCommand.type, "trigger"),
      eq(deviceCommand.status, "acked"),
      gte(deviceCommand.createdAt, since),
    );
  const trig = () => db.select().from(deviceCommand).innerJoin(deviceTable, eq(deviceCommand.deviceId, deviceTable.id));

  const [dailyRows, monthlyRows, gridRows] = await Promise.all([
    db.select({ bucket: dayExpr, count: count() }).from(deviceCommand).innerJoin(deviceTable, eq(deviceCommand.deviceId, deviceTable.id)).where(scoped(since30)).groupBy(dayExpr),
    db.select({ bucket: monthExpr, count: count() }).from(deviceCommand).innerJoin(deviceTable, eq(deviceCommand.deviceId, deviceTable.id)).where(scoped(since9mo)).groupBy(monthExpr),
    db.select({ dow: dowExpr, hour: hourExpr, count: count() }).from(deviceCommand).innerJoin(deviceTable, eq(deviceCommand.deviceId, deviceTable.id)).where(scoped(since90)).groupBy(sql`1`, sql`2`),
  ]);
```

(Delete the unused `trig` helper line if you prefer inlining — shown inline above; remove the `const trig` line.)

- [ ] **Step 4: In `getStoresAnalytics`, repoint the cross-store month query.** Replace lines ~518–524:

```ts
    const monthExpr = sql<string>`to_char(date_trunc('month', ${documentTable.createdAt}), 'YYYY-MM')`;

    const perStoreMonth = await db
      .select({ storeId: documentTable.storeId, bucket: monthExpr, count: count() })
      .from(documentTable)
      .where(and(eq(documentTable.organizationId, organizationId), gte(documentTable.createdAt, since9mo)))
      .groupBy(documentTable.storeId, monthExpr);
```

with:

```ts
    const monthExpr = sql<string>`to_char(date_trunc('month', ${deviceCommand.createdAt}), 'YYYY-MM')`;

    const perStoreMonth = await db
      .select({ storeId: deviceTable.storeId, bucket: monthExpr, count: count() })
      .from(deviceCommand)
      .innerJoin(deviceTable, eq(deviceCommand.deviceId, deviceTable.id))
      .where(and(
        eq(deviceCommand.organizationId, organizationId),
        eq(deviceCommand.type, "trigger"),
        eq(deviceCommand.status, "acked"),
        gte(deviceCommand.createdAt, since9mo),
      ))
      .groupBy(deviceTable.storeId, monthExpr);
```

- [ ] **Step 5: Build + typecheck.** Run: `npm run build`. Expected: PASS (no type errors; `documentTable` may now be unused in these functions but is still used by the untouched billing/health code, so no unused-import error yet).

- [ ] **Step 6: Write the read-only verification script.** Create `<scratchpad>/verify-task1.mjs`:

```js
import './lib/db/load-env.ts';
import { neon } from '@neondatabase/serverless';
import { getTenantDashboard } from './lib/data.ts';
const sql = neon(process.env.DATABASE_URL);
// Find an org that has acked triggers this month.
const orgRow = (await sql.query(
  `SELECT organization_id AS org, count(*)::int AS c
   FROM device_command
   WHERE type='trigger' AND status='acked'
     AND created_at >= date_trunc('month', now())
   GROUP BY organization_id ORDER BY c DESC LIMIT 1`
))[0];
if (!orgRow) { console.log('No acked triggers this month — dashboard will show 0 (expected).'); process.exit(0); }
const raw = orgRow.c;
const dash = await getTenantDashboard(orgRow.org);
console.log(`org=${orgRow.org} raw acked-triggers-this-month=${raw} dashboard.documentsThisMonth=${dash.documentsThisMonth}`);
if (dash.documentsThisMonth !== raw) { console.error('MISMATCH'); process.exit(1); }
console.log('MATCH ✓');
```

- [ ] **Step 7: Run the verification.** Run: `npx tsx <scratchpad>/verify-task1.mjs`. Expected: `MATCH ✓` (or the "no acked triggers this month" message if the 2 prod triggers predate the current month — in that case additionally run a 9-month-window cross-check by querying `getStoresAnalytics` and comparing to raw counts over `since9mo`). Then delete the script: `rm <scratchpad>/verify-task1.mjs`.

- [ ] **Step 8: Commit.**

```bash
git add lib/data.ts
git commit -m "feat(metrics): repoint tenant+store dashboards from document table to acked triggers"
```

---

### Task 2: Repoint platform health + alerts to the trigger status model

Health tracked the document lifecycle (`ready/downloaded/pending`). Triggers use `pending/delivered/acked/failed/expired`, so the health card's shape changes, not just its source. Repoint `getPlatformHealth`, `getAlertInputs`, `getCustomerDetail`; reshape the `PlatformHealth.ingest` block to a trigger `activity` block; update `zeroedHealth()` and the health UI.

**Files:**
- Modify: `lib/data.ts` — `PlatformHealth` interface, `zeroedHealth()`, `getPlatformHealth` (~1310–1351), `getAlertInputs` (~1415–1432), `getCustomerDetail` (~709–723)
- Modify: `app/(admin)/admin/health/page.tsx` (~44–86)
- Modify: `app/(admin)/admin/customers/[tenantId]/page.tsx:134` ("Stuck docs" label)
- Verify (read-only, scratchpad): `<scratchpad>/verify-task2.mjs`

**Interfaces:**
- Consumes: `deviceCommand`, `deviceTable`, `orgTable`, `count/and/eq/gte/lt/max/desc`, `STUCK_PENDING_MINUTES`, `computeAlerts`.
- Produces: new `PlatformHealth.activity` shape `{ last1h: number; last24h: number; acked: number; pending: number; failed: number; stuckPending: number }` (replaces `ingest`); `PlatformHealth.usage.inactiveTenants[].lastActivityAt: string | null` (replaces `lastDocumentAt`). `getAlertInputs`/`getCustomerDetail` return types unchanged (still `stuckPendingCount`, `lastActivityAt` already the field name in `getCustomerDetail`).

- [ ] **Step 1: Find the `PlatformHealth` interface + `zeroedHealth()` in `lib/data.ts`.** Run: `grep -n "interface PlatformHealth\|function zeroedHealth\|ingest:\|lastDocumentAt" lib/data.ts`. Note the exact line ranges of the `ingest` block in the type and in `zeroedHealth()`.

- [ ] **Step 2: Rename the `ingest` type block to `activity` with trigger subfields.** In the `PlatformHealth` interface, replace the `ingest: { last1h; last24h; ready; downloaded; pending; stuckPending }` block with:

```ts
  activity: {
    last1h: number;
    last24h: number;
    acked: number;
    pending: number;
    failed: number;
    stuckPending: number;
  };
```

and in the `inactiveTenants` item type change `lastDocumentAt: string | null` → `lastActivityAt: string | null`.

- [ ] **Step 3: Update `zeroedHealth()`** to return the new shape: replace its `ingest: { last1h: 0, last24h: 0, ready: 0, downloaded: 0, pending: 0, stuckPending: 0 }` with `activity: { last1h: 0, last24h: 0, acked: 0, pending: 0, failed: 0, stuckPending: 0 }`. (No `inactiveTenants` entries exist in the zero case, so no field rename needed there.)

- [ ] **Step 4: Repoint the `getPlatformHealth` metric queries.** Replace lines ~1310–1351 (the `last1h`/`last24h`/`breakdownRows`/`stuckPending`/`topRows`/`lastRows`/`inactiveTenants` block) with:

```ts
    const trigAcked = and(eq(deviceCommand.type, "trigger"), eq(deviceCommand.status, "acked"));
    const [{ last1h }] = await db.select({ last1h: count() }).from(deviceCommand).where(and(trigAcked, gte(deviceCommand.createdAt, h1)));
    const [{ last24h }] = await db.select({ last24h: count() }).from(deviceCommand).where(and(trigAcked, gte(deviceCommand.createdAt, h24)));
    // Status breakdown over all trigger commands in 24h (not just acked).
    const breakdownRows = await db
      .select({ status: deviceCommand.status, c: count() })
      .from(deviceCommand)
      .where(and(eq(deviceCommand.type, "trigger"), gte(deviceCommand.createdAt, h24)))
      .groupBy(deviceCommand.status);
    const bd = { acked: 0, pending: 0, failed: 0 } as Record<string, number>;
    for (const r of breakdownRows) {
      if (r.status === "acked") bd.acked += Number(r.c);
      else if (r.status === "pending" || r.status === "delivered") bd.pending += Number(r.c);
      else bd.failed += Number(r.c); // failed + expired
    }
    const [{ stuckPending }] = await db
      .select({ stuckPending: count() })
      .from(deviceCommand)
      .where(and(eq(deviceCommand.type, "trigger"), eq(deviceCommand.status, "pending"), lt(deviceCommand.createdAt, stuckCut)));

    const topRows = await db
      .select({ id: orgTable.id, name: orgTable.name, c: count() })
      .from(deviceCommand)
      .innerJoin(orgTable, eq(deviceCommand.organizationId, orgTable.id))
      .where(and(trigAcked, gte(deviceCommand.createdAt, h24)))
      .groupBy(orgTable.id, orgTable.name)
      .orderBy(desc(count()))
      .limit(5);
    const topTenants = topRows.map((r) => ({ id: r.id, name: r.name, count: Number(r.c) }));

    const allOrgs = await db.select({ id: orgTable.id, name: orgTable.name }).from(orgTable);
    const lastRows = await db
      .select({ org: deviceCommand.organizationId, last: max(deviceCommand.createdAt) })
      .from(deviceCommand)
      .where(trigAcked)
      .groupBy(deviceCommand.organizationId);
    const lastByOrg = new Map<string, Date>();
    for (const r of lastRows) if (r.last) lastByOrg.set(r.org, r.last);
    const inactiveTenants = allOrgs
      .filter((o) => {
        const last = lastByOrg.get(o.id);
        return !last || last < inactiveCut;
      })
      .map((o) => ({
        id: o.id,
        name: o.name,
        lastActivityAt: lastByOrg.get(o.id)?.toISOString() ?? null,
      }));
```

- [ ] **Step 5: Update the `getPlatformHealth` return block** — replace the `ingest: { ... }` object with:

```ts
      activity: {
        last1h: Number(last1h),
        last24h: Number(last24h),
        acked: bd.acked,
        pending: bd.pending,
        failed: bd.failed,
        stuckPending: Number(stuckPending),
      },
```

(The `usage: { topTenants, inactiveTenants }` line is unchanged; `inactiveTenants` now carries `lastActivityAt`.)

- [ ] **Step 6: Repoint `getAlertInputs`** — replace its `stuckPendingCount` and `lastRows` document queries (lines ~1415–1432) with the trigger equivalents:

```ts
  const [{ stuckPendingCount }] = await db
    .select({ stuckPendingCount: count() })
    .from(deviceCommand)
    .where(and(eq(deviceCommand.type, "trigger"), eq(deviceCommand.status, "pending"), lt(deviceCommand.createdAt, stuckCut)));

  const allOrgs = await db.select({ id: orgTable.id, name: orgTable.name }).from(orgTable);
  const lastRows = await db
    .select({ org: deviceCommand.organizationId, last: max(deviceCommand.createdAt) })
    .from(deviceCommand)
    .where(and(eq(deviceCommand.type, "trigger"), eq(deviceCommand.status, "acked")))
    .groupBy(deviceCommand.organizationId);
```

(The `lastByOrg`/`inactiveTenants` mapping below is unchanged — it only reads `{id,name}`.)

- [ ] **Step 7: Repoint `getCustomerDetail`'s stuck + last-activity queries** — replace lines ~710–723:

```ts
  const [{ stuck }] = await db
    .select({ stuck: sql<number>`count(*)::int` })
    .from(deviceCommand)
    .where(
      and(
        eq(deviceCommand.organizationId, organizationId),
        eq(deviceCommand.type, "trigger"),
        eq(deviceCommand.status, "pending"),
        lt(deviceCommand.createdAt, stuckCutoff),
      ),
    );
  const [{ last }] = await db
    .select({ last: max(deviceCommand.createdAt) })
    .from(deviceCommand)
    .where(and(
      eq(deviceCommand.organizationId, organizationId),
      eq(deviceCommand.type, "trigger"),
      eq(deviceCommand.status, "acked"),
    ));
```

(`stuck`/`last` feed `tenantHealthLevel` inputs `stuckPendingCount`/`lastActivityAt` — names unchanged.)

- [ ] **Step 8: Update the health UI.** In `app/(admin)/admin/health/page.tsx` replace the "Ingest activity" section (~44–56):

```tsx
      <section className="flex flex-col gap-3">
        <h2 className="text-lg font-medium">Trigger activity</h2>
        <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
          <KpiCard label="Activations (1h)" value={String(h.activity.last1h)} />
          <KpiCard label="Activations (24h)" value={String(h.activity.last24h)} />
          <KpiCard label="Stuck pending" value={String(h.activity.stuckPending)} />
        </div>
        <p className="text-sm text-muted-foreground">
          Last 24h: {h.activity.acked} acked · {h.activity.pending} pending · {h.activity.failed} failed
        </p>
      </section>
```

and update the two "No documents in the last 24h." → "No activations in the last 24h." string and `t.lastDocumentAt` → `t.lastActivityAt` (both occurrences in that file).

- [ ] **Step 9: Update the customer-detail health label.** In `app/(admin)/admin/customers/[tenantId]/page.tsx:134`, change the tile label `"Stuck docs"` → `"Stuck pending"`. (The value `health.stuckPendingCount` field name is unchanged.)

- [ ] **Step 10: Build.** Run: `npm run build`. Expected: PASS. Fix any remaining `h.ingest`/`lastDocumentAt` references the compiler flags.

- [ ] **Step 11: Verify (read-only).** Create `<scratchpad>/verify-task2.mjs`:

```js
import './lib/db/load-env.ts';
import { neon } from '@neondatabase/serverless';
import { getPlatformHealth } from './lib/data.ts';
const sql = neon(process.env.DATABASE_URL);
const raw24 = (await sql.query(
  `SELECT count(*)::int c FROM device_command WHERE type='trigger' AND status='acked' AND created_at >= now() - interval '24 hours'`
))[0].c;
const h = await getPlatformHealth();
console.log(`raw acked-24h=${raw24}  health.activity.last24h=${h.activity.last24h}`);
console.log('activity block:', JSON.stringify(h.activity));
if (h.activity.last24h !== raw24) { console.error('MISMATCH'); process.exit(1); }
console.log('MATCH ✓');
```

Run: `npx tsx <scratchpad>/verify-task2.mjs`. Expected: `MATCH ✓`. Then `rm <scratchpad>/verify-task2.mjs`.

- [ ] **Step 12: Run existing tests.** Run: `npm test`. Expected: PASS — `health.test.ts` (`computeAlerts`) and `alerts.test.ts` inputs are unchanged, `tenant-health.test.ts` inputs (`stuckPendingCount`, `lastActivityAt`) unchanged.

- [ ] **Step 13: Commit.**

```bash
git add lib/data.ts "app/(admin)/admin/health/page.tsx" "app/(admin)/admin/customers/[tenantId]/page.tsx"
git commit -m "feat(health): repoint platform health + alerts to trigger status model"
```

---

### Task 3: Rename metric vocabulary `documents` → `activations` (cosmetic)

One atomic rename across the pure helpers, `data.ts` view models, and UI. No behavior change. Gated by the existing Vitest suites (pure helpers) + build. Must be atomic (a partial rename breaks the build). **Do NOT touch billing fields** (`Invoice.documents`, `invoice.documentCount`) — those are a different concept.

**Files (all):**
- Modify + Test: `lib/analytics.ts`, `lib/analytics.test.ts`
- Modify + Test: `lib/eco.ts`, and any eco test (`grep -rl "computeEcoSavings\|EcoSavings" lib/*.test.ts`)
- Modify: `lib/types.ts`, `lib/data.ts`
- Modify (UI): the files enumerated in Step 5

**Interfaces:**
- Renames (identifiers): `documentsToday→activationsToday`, `documentsThisMonth→activationsThisMonth`, `TimePoint.documents→TimePoint.activations`, `EcoSavings.documents→EcoSavings.activations`, `ecoYtdDocuments→ecoYtdActivations`, `StoreComparisonRow.documentsThisMonth→activationsThisMonth`, `computeEcoSavings(documents)` param → `activations`.
- **NOT renamed:** `Invoice.documents`, `invoice.documentCount`, `document`/`documentTable` table symbol, `usageEvent`, anything under `lib/billing*`.

- [ ] **Step 1 (TDD): Update pure-helper test expectations first.** In `lib/analytics.test.ts`, rename every `documents` property in expected objects to `activations` and every `documentsThisMonth` to `activationsThisMonth` (e.g. `bucketsToSeries` result assertions `{ label, documents, revenue }` → `{ label, activations, revenue }`; `toComparisonRows` `documentsThisMonth` → `activationsThisMonth`). In the eco test (from the grep), rename `EcoSavings.documents` assertions → `activations`.

- [ ] **Step 2: Run the tests to watch them fail.** Run: `npm test -- analytics eco`. Expected: FAIL (source still emits `documents`).

- [ ] **Step 3: Rename in `lib/analytics.ts`.** Change `bucketsToSeries` local `const documents` → `const activations` and returned `{ label, documents, revenue }` → `{ label, activations, revenue }`; `Heatmap`/`StoreComparisonRow` doc-comments "document count" → "activation count"; `StoreComparisonRow.documentsThisMonth` → `activationsThisMonth`; in `toComparisonRows` the `documentsThisMonth: s.current` → `activationsThisMonth: s.current` and the `.sort((a,b)=> b.documentsThisMonth - a.documentsThisMonth)` → `activationsThisMonth`.

- [ ] **Step 4: Rename in `lib/eco.ts` and `lib/types.ts`.**
  - `lib/eco.ts`: `EcoSavings.documents` → `activations`; `computeEcoSavings(documents: number)` param → `activations` and its body references. **Keep the `PAPER_GRAMS_PER_DOCUMENT` constant names and the eco *user-facing* copy as-is** (eco "documents" = sheets of paper avoided — a deliberate exception; only the field/param renames).
  - `lib/types.ts`: line ~21–22 `documentsToday/documentsThisMonth` on `Device`; `TimePoint.documents` (line ~54); `StoreSummary.documentsThisMonth` (~85); `Tenant.documentsThisMonth` (~102). **Leave `Invoice.documents` (~66) unchanged** — billing.

- [ ] **Step 5: Rename in `lib/data.ts` producers.** Rename every assignment/return of the renamed fields: `mapDevice` (`documentsToday/documentsThisMonth`), `buildTenant`/`summarize` (`documentsThisMonth`), `dailySeries/monthlySeries/sumSeries` (`documents` on `TimePoint`), `getTenantDashboard` (`documentsToday`, `documentsThisMonth`, `ecoYtdDocuments`→`ecoYtdActivations`), `getStoreAnalytics`/`getStoresAnalytics` (`.documents` reads like `monthly[...]?.documents` → `.activations`), `getTenantStores`, `getTenantSummaries`, `getAllDevices`, `getAdminOverview`. Run `grep -n "documentsToday\|documentsThisMonth\|\.documents\b\|ecoYtdDocuments" lib/data.ts` and change each **except** any `invoice`/billing context. Verify with `npm run build` after.

- [ ] **Step 6: Rename in the UI.** Apply these exact edits (field reads + labels). Run `npm run build` after — the compiler will flag any missed field read.

  - `app/(tenant)/tenant/page.tsx`: `documentsThisMonth`→`activationsThisMonth` (sort + store row); `dash.documentsToday`→`dash.activationsToday`; `dash.documentsThisMonth`→`dash.activationsThisMonth`; KPI labels "Documents today/this month"→"Activations today/this month"; "Daily digital documents, last 30 days"→"Daily activations, last 30 days".
  - `app/(tenant)/tenant/stores/page.tsx:78`: `s.documentsThisMonth`→`s.activationsThisMonth`.
  - `app/(tenant)/tenant/stores/[storeId]/page.tsx`: `documentsToday/documentsThisMonth` reads → `activations*`; hint "documents this month"→"activations this month"; "Daily digital documents, last 30 days"→"Daily activations, last 30 days"; "documents at this store."→"activations at this store."
  - `app/(tenant)/tenant/stores/[storeId]/[deviceId]/page.tsx:69,74`: `device.documentsToday/ThisMonth`→`activations*`.
  - `app/(admin)/admin/page.tsx`: `o.documentsThisMonth`/`c.documentsThisMonth`→`activationsThisMonth`; "Monthly documents, all customers"→"Monthly activations, all customers".
  - `app/(admin)/admin/customers/page.tsx:83`: `c.documentsThisMonth`→`c.activationsThisMonth`.
  - `app/(admin)/admin/customers/[tenantId]/page.tsx:68,145,271`: `d.documentsThisMonth`/`summary.documentsThisMonth`→`activationsThisMonth`.
  - `app/(admin)/admin/devices/[deviceId]/page.tsx:75-76`: labels "Documents today/this month"→"Activations today/this month"; `device.documentsToday/ThisMonth`→`activations*`.
  - `app/(admin)/admin/devices/page.tsx:31`: hint "accepting documents"→"ready to trigger".
  - `components/fleet-table.tsx:137`: `r.documentsThisMonth`→`r.activationsThisMonth`.
  - `components/device-card.tsx:69,75`: `device.documentsToday/ThisMonth`→`activations*`.
  - `app/(tenant)/tenant/analytics/page.tsx`: `r.documentsThisMonth`→`r.activationsThisMonth`; "Compare document volume…"→"Compare activation volume…"; "Once your stores start issuing documents…"→"Once your stores start showing QR codes…"; "{n} documents · {rev}"→"{n} activations · {rev}"; "Monthly documents per store, last 9 months"→"Monthly activations per store, last 9 months".
  - `app/(tenant)/tenant/reports/page.tsx`: `s.documentsThisMonth`/`d.documentsThisMonth`→`activationsThisMonth`; `p.documents`→`p.activations`; `totalDocuments`→`totalActivations`; CSV header cells "documents"→"activations"; "Monthly digital documents, last 9 months"→"Monthly activations, last 9 months"; "Top printers by documents this month"→"Top printers by activations this month". (Its `p.documents`→paper-kg eco calc still works; eco field is now `activations`.)
  - `components/charts.tsx`: `dataKey="documents"`→`dataKey="activations"` (both chart configs, ~92,224,266); `unit="documents"`→`unit="activations"` (~97); `row[s.storeId] = s.monthly[i]?.documents`→`.activations` (~275); comment "monthly documents comparison"→"monthly activations comparison".
  - `components/eco-savings.tsx:58,60`: `eco.documents`→`eco.activations`; **keep** the visible copy "paperless documents {period}." (paper-avoided semantics — the deliberate eco exception).
  - `components/peak-heatmap.tsx:26,28,36-37,61`: "No documents in the last 90 days yet"→"No activations in the last 90 days yet"; "…once this store starts issuing documents."→"…once this store starts showing QR codes."; aria "Document activity heatmap…{total} documents total."→"Activation heatmap…{total} activations total."; cell title "{count} document(s)"→"{count} activation(s)".
  - `app/(admin)/admin/billing/page.tsx:137`: `t.documentsThisMonth`→`t.activationsThisMonth` (metric field only — leave `inv.documents`/`inv.documentCount` and the per-print copy untouched).

- [ ] **Step 7: Run tests + build.** Run: `npm test` then `npm run build`. Expected: BOTH PASS. (`analytics.test.ts`/eco test now green with renamed fields; billing tests untouched and green.)

- [ ] **Step 8: Grep for stragglers.** Run: `grep -rn "documentsToday\|documentsThisMonth\|ecoYtdDocuments" app/ components/ lib/ | grep -v "invoice\|billing\|Invoice\|documentCount"`. Expected: no output. If any remain, rename them and rebuild.

- [ ] **Step 9: Commit.**

```bash
git add lib/ app/ components/
git commit -m "refactor(metrics): rename document metric vocabulary to activations"
```

---

## Rollout (after all tasks reviewed + merged to main)

- Code-only, no migration. Deploy: `vercel --prod --yes` (git auto-deploy is disconnected — see [[vercel-deploy]] memory).
- Post-deploy read-only smoke: the ~2 real acked triggers on prod should now surface on the tenant dashboard / admin health page instead of the frozen 33. Sign in as `dana@roastwell.co` and confirm the dashboard "Activations this month" and the admin health "Trigger activity" card reflect live data.

## Accepted transitional divergence (documented, not a bug)

Billing stays document/per-print based (deferred follow-up #2), so the billing page's `documentCount` may differ from the dashboards' activation counts until billing is retired. This is intentional. If it causes operator confusion before #2 lands, add a one-line note to the billing page — not in scope here.

## Self-review notes

- **Spec coverage:** metric unit ✓ (Task 1/2 filters), store attribution join ✓ (Task 1 S3/S4, Task 2), naming ✓ (Task 3), health semantic remap ✓ (Task 2), eco/revenue keep-formula-swap-source ✓ (Task 1 inherits via loadOrg; eco field rename Task 3), billing out-of-scope guardrails ✓ (Global Constraints + Task 3 exclusions), read-only-prod verification ✓ (Global Constraints + verify scripts), no migration ✓.
- **Placeholder scan:** none — every step carries exact code or exact old→new strings.
- **Type consistency:** `activity` shape defined once (Task 2 Interfaces) and consumed in Task 2 S5/S8; `activationsThisMonth`/`activationsToday`/`TimePoint.activations` defined in types.ts (Task 3 S4) and consumed consistently in S5/S6.
