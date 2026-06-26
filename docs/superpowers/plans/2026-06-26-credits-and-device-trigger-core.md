# Credits + Device-Trigger Core (Spec A) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A public `POST /api/v1/devices/{id}/trigger` endpoint that enqueues a generic action (first: `show_qr`) to one of the caller's devices, gated by a prepaid credit balance (reserve→settle→release) and a `devices:trigger` API-key scope, with admin + Stripe top-up and per-company/per-device analytics — all off an append-only credit ledger.

**Architecture:** Append-only `credit_ledger` (source of truth + analytics) + a `credit_balance` cache, mutated only through **single-statement atomic SQL guards** (Neon `neon-http` has no interactive transactions). A trigger reserves a hold; the device's existing ack endpoint settles it; a cron releases expired holds. API keys gain capability scopes. Stripe one-time Checkout tops up credits via the existing webhook.

**Tech Stack:** Next.js 16 App Router, Drizzle over `neon-http`, Stripe, Vitest (pure-function tests only — DB code verified by build + a throwaway integration script).

## Global Constraints

- **`CREDITS_PER_TRIGGER = 1`**; cost resolved per action via a registry (`show_qr` = 1).
- **Command TTL = 60s** (`expiresAt = now + 60_000`).
- **`Idempotency-Key` header REQUIRED** on the trigger POST (400 if missing). Replays return the stored response; never reserve twice.
- **Atomicity rule:** every balance mutation is ONE SQL statement with a `WHERE` guard + `RETURNING`. No interactive transactions. If a guard returns no row → the precondition failed (insufficient / wrong state) and nothing changed.
- **Credit-move idempotency:** the `deviceCommand` status transition is the lock — `UPDATE device_command SET status=? WHERE id=? AND status=? RETURNING id`; only the winner moves credits. Double-ack or ack-vs-cron races can't double-move.
- **Scopes:** known scopes `receipts:read`, `usage:read`, `devices:trigger`. New keys default to `["receipts:read","usage:read"]`. **Existing keys are backfilled to `["receipts:read","usage:read"]` — never `devices:trigger`.**
- **Device eligibility:** trigger accepted only when `effectiveDeviceStatus(device.status, device.lastSeenAt, now)` === `"online"`; else 409, no charge.
- **No receipt row** for a trigger. The device only ENCODES the url into a QR (no device fetch).
- **Stripe purchase replay-safe:** `purchase` ledger row keyed by Stripe `session.id` under a unique `(kind, idempotency_key)` partial index; insert ledger row BEFORE bumping balance.
- **Test reality:** pure logic is TDD'd in `lib/**/*.test.ts` (vitest, no DB). DB primitives/endpoints are verified by `npm run build` + a throwaway `lib/db/_*.ts` integration script run against the dev DB (which IS prod — scripts must be read-mostly and self-cleaning; only ever touch a dedicated test org/device).
- **ID prefixes:** `id("cl")` ledger, `id("cmd")` command, `id("ak")` api key, `id("aud")` audit. API key hashing: `hashApiKey` = SHA-256 hex.
- Branch: `feat/credits-device-trigger` (already created). Commit footer: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.

---

## File Structure

- `lib/db/schema.ts` — `creditBalance`, `creditLedger`, `apiIdempotency` tables; `apiKey.scopes`; `deviceCommand.action/payload/expiresAt` + `"trigger"` type + `"expired"` status. (Task 1)
- `drizzle/NNNN_*.sql` — generated migration, hand-edited to backfill scopes. (Task 1)
- `lib/api-scopes.ts` (+ test) — scope constants, `hasScope`, `DEFAULT_KEY_SCOPES`, `sanitizeScopes`. (Task 2)
- `lib/trigger-actions.ts` (+ test) — action registry, `creditCostForAction`, `validateTriggerBody`. (Task 2)
- `lib/credits.ts` — atomic primitives `reserveCredit`/`settleHold`/`releaseHold`/`grantCredits`/`getBalance`. (Task 3)
- `app/api/v1/devices/[deviceId]/trigger/route.ts` — the trigger endpoint. (Task 4)
- `app/api/device/commands/ack/route.ts` — extend: guarded transition + settle/release. (Task 5)
- `app/api/device/commands/route.ts` — include `action`+`payload` for trigger commands. (Task 5)
- `app/api/cron/credit-holds/route.ts` + `vercel.json` — release expired holds + idempotency cleanup. (Task 5)
- `lib/actions/api-keys.ts` + `components/api-key-create-dialog.tsx` — scope picker. (Task 6)
- `lib/actions/credits.ts` + admin customer page — credit grant + balance/ledger view. (Task 7)
- `lib/billing/credit-packs.ts`, `lib/billing/stripe-billing.ts`, `app/api/stripe/webhook/route.ts`, tenant billing UI — Stripe purchase. (Task 8)
- `lib/data.ts` (+ analytics test for pure mappers) — credit usage queries; surface via `/api/v1/usage` + UI. (Task 9)

---

## Task 1: Schema + migration (tables, columns, scopes backfill)

**Files:**
- Modify: `lib/db/schema.ts`
- Modify: `lib/device-commands.ts` (add `"trigger"` to `COMMAND_TYPES`)
- Create: `drizzle/<generated>.sql` (then hand-edit)

**Interfaces:**
- Produces: tables `credit_balance`, `credit_ledger`, `api_idempotency`; `apiKey.scopes text[]`; `deviceCommand.action/payload/expiresAt`, type incl `"trigger"`, status incl `"expired"`.

- [ ] **Step 1: Add tables + columns to `lib/db/schema.ts`**

After the `apiKey` table (line 322), add `scopes` to apiKey by editing its columns block to include:
```ts
    scopes: text("scopes").array().notNull().default(sql`'{}'::text[]`),
```
(Import `sql` from `drizzle-orm` if not already imported in this file.)

Extend `deviceCommand` (lines 277–292): add to the `type` enum list `"trigger"`, to the `status` enum list `"expired"`, and add columns:
```ts
    action: text("action"),
    payload: jsonb("payload"),
    expiresAt: timestamp("expires_at"),
```
(Ensure `jsonb` is imported from `drizzle-orm/pg-core`.)

Add three new tables (near the other app tables):
```ts
export const creditBalance = pgTable("credit_balance", {
  organizationId: text("organization_id").primaryKey().references(() => organization.id, { onDelete: "cascade" }),
  available: integer("available").notNull().default(0),
  held: integer("held").notNull().default(0),
  updatedAt: timestamp("updated_at").$defaultFn(() => new Date()).notNull(),
});

export const creditLedger = pgTable(
  "credit_ledger",
  {
    id: text("id").primaryKey(),
    organizationId: text("organization_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
    deviceId: text("device_id").references(() => device.id, { onDelete: "set null" }),
    kind: text("kind", { enum: ["grant", "purchase", "hold", "settle", "release"] }).notNull(),
    credits: integer("credits").notNull(),
    action: text("action"),
    commandId: text("command_id"),
    idempotencyKey: text("idempotency_key"),
    balanceAfterAvailable: integer("balance_after_available"),
    note: text("note"),
    createdByUserId: text("created_by_user_id"),
    createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  },
  (t) => [
    index("credit_ledger_org_created_idx").on(t.organizationId, t.createdAt),
    index("credit_ledger_device_created_idx").on(t.deviceId, t.createdAt),
    index("credit_ledger_command_idx").on(t.commandId),
    uniqueIndex("credit_ledger_kind_idem_idx").on(t.kind, t.idempotencyKey).where(sql`${t.idempotencyKey} is not null`),
  ],
);

export const apiIdempotency = pgTable(
  "api_idempotency",
  {
    key: text("key").notNull(),
    organizationId: text("organization_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
    responseStatus: integer("response_status").notNull(),
    responseBody: jsonb("response_body").notNull(),
    commandId: text("command_id"),
    createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  },
  (t) => [primaryKey({ columns: [t.key, t.organizationId] })],
);
```
(Import `integer`, `jsonb`, `primaryKey`, `uniqueIndex`, `index` from `drizzle-orm/pg-core` as needed — most are already imported.)

In `lib/device-commands.ts`, add `"trigger"` to `COMMAND_TYPES`:
```ts
export const COMMAND_TYPES = ["reboot", "refresh", "identify", "config-changed", "firmware-update", "trigger"] as const;
```

- [ ] **Step 2: Generate the migration**

Run: `npm run db:generate`
Expected: a new `drizzle/NNNN_*.sql` creating the 3 tables, the `api_key.scopes` column, and the `device_command` columns. `tsc` clean for schema.ts.

- [ ] **Step 3: Hand-edit the migration to backfill existing API-key scopes**

Open the generated SQL and append (after the `ADD COLUMN ... scopes` line):
```sql
UPDATE "api_key" SET "scopes" = ARRAY['receipts:read','usage:read'] WHERE "scopes" = '{}'::text[];
```
This preserves existing keys' read ability without granting `devices:trigger`.

- [ ] **Step 4: Apply to the dev DB + sanity check**

Run: `npm run db:migrate`
Then verify with a throwaway script `lib/db/_check_schema.ts`:
```ts
import "./load-env";
import { db } from "../db";
import { apiKey, creditBalance } from "./schema";
async function main() {
  const keys = await db.select({ id: apiKey.id, scopes: apiKey.scopes }).from(apiKey).limit(5);
  console.log("sample key scopes:", keys.map((k) => k.scopes));
  await db.select().from(creditBalance).limit(1); // table exists
  console.log("credit_balance table OK");
  process.exit(0);
}
main();
```
Run `npx tsx lib/db/_check_schema.ts` (expect existing keys show `["receipts:read","usage:read"]`), then `rm lib/db/_check_schema.ts`.

- [ ] **Step 5: Commit**

```bash
git add lib/db/schema.ts lib/device-commands.ts drizzle/
git commit -m "feat(credits): schema for balance/ledger/idempotency, key scopes, trigger command

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Pure helpers — scopes + trigger-action validation (TDD)

**Files:**
- Create: `lib/api-scopes.ts` + `lib/api-scopes.test.ts`
- Create: `lib/trigger-actions.ts` + `lib/trigger-actions.test.ts`

**Interfaces:**
- Produces: `API_SCOPES`, `ApiScope`, `DEFAULT_KEY_SCOPES`, `hasScope(scopes, required)`, `sanitizeScopes(raw)`; `TRIGGER_ACTIONS`, `creditCostForAction(action)`, `validateTriggerBody(raw) → { ok:true, action, payload } | { ok:false, error }`.

- [ ] **Step 1: Write failing tests — scopes**

`lib/api-scopes.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { hasScope, sanitizeScopes, DEFAULT_KEY_SCOPES, API_SCOPES } from "./api-scopes";

describe("api scopes", () => {
  it("DEFAULT_KEY_SCOPES is read-only (no devices:trigger)", () => {
    expect(DEFAULT_KEY_SCOPES).toEqual(["receipts:read", "usage:read"]);
    expect(DEFAULT_KEY_SCOPES).not.toContain("devices:trigger");
  });
  it("hasScope is true only when present", () => {
    expect(hasScope(["devices:trigger"], "devices:trigger")).toBe(true);
    expect(hasScope(["receipts:read"], "devices:trigger")).toBe(false);
    expect(hasScope([], "devices:trigger")).toBe(false);
  });
  it("sanitizeScopes keeps only known scopes, dedupes, drops junk", () => {
    expect(sanitizeScopes(["devices:trigger", "devices:trigger", "nope", 5 as never]))
      .toEqual(["devices:trigger"]);
    expect(sanitizeScopes("notanarray" as never)).toEqual([]);
    expect(new Set(API_SCOPES).has("usage:read")).toBe(true);
  });
});
```

`lib/trigger-actions.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { validateTriggerBody, creditCostForAction } from "./trigger-actions";

describe("validateTriggerBody", () => {
  it("accepts show_qr with an https url", () => {
    const r = validateTriggerBody({ action: "show_qr", payload: { url: "https://x.co/r/abc" } });
    expect(r).toEqual({ ok: true, action: "show_qr", payload: { url: "https://x.co/r/abc" } });
  });
  it("rejects unknown action", () => {
    expect(validateTriggerBody({ action: "explode", payload: {} }).ok).toBe(false);
  });
  it("rejects show_qr without a valid url", () => {
    expect(validateTriggerBody({ action: "show_qr", payload: {} }).ok).toBe(false);
    expect(validateTriggerBody({ action: "show_qr", payload: { url: "ftp://x" } }).ok).toBe(false);
    expect(validateTriggerBody({ action: "show_qr", payload: { url: "x".repeat(3000) } }).ok).toBe(false);
  });
  it("rejects non-object body", () => {
    expect(validateTriggerBody(null).ok).toBe(false);
    expect(validateTriggerBody("nope").ok).toBe(false);
  });
  it("creditCostForAction returns 1 for show_qr", () => {
    expect(creditCostForAction("show_qr")).toBe(1);
  });
});
```

- [ ] **Step 2: Run tests, verify fail**

Run: `npm run test -- api-scopes trigger-actions`
Expected: FAIL (modules not found).

- [ ] **Step 3: Implement `lib/api-scopes.ts`**
```ts
export const API_SCOPES = ["receipts:read", "usage:read", "devices:trigger"] as const;
export type ApiScope = (typeof API_SCOPES)[number];
export const DEFAULT_KEY_SCOPES: ApiScope[] = ["receipts:read", "usage:read"];

export function hasScope(scopes: readonly string[] | null | undefined, required: ApiScope): boolean {
  return Array.isArray(scopes) && scopes.includes(required);
}

/** Keep only known scopes, de-duplicated, preserving canonical order. */
export function sanitizeScopes(raw: unknown): ApiScope[] {
  if (!Array.isArray(raw)) return [];
  const set = new Set(raw.filter((s): s is string => typeof s === "string"));
  return API_SCOPES.filter((s) => set.has(s));
}
```

- [ ] **Step 4: Implement `lib/trigger-actions.ts`**
```ts
export const TRIGGER_ACTIONS = ["show_qr"] as const;
export type TriggerAction = (typeof TRIGGER_ACTIONS)[number];

const COST: Record<TriggerAction, number> = { show_qr: 1 };
export function creditCostForAction(action: TriggerAction): number {
  return COST[action];
}

const MAX_URL = 2048;
export type TriggerBody = { action: TriggerAction; payload: Record<string, unknown> };
export type ValidateResult = { ok: true; action: TriggerAction; payload: Record<string, unknown> } | { ok: false; error: string };

function isValidUrl(u: unknown): u is string {
  if (typeof u !== "string" || u.length === 0 || u.length > MAX_URL) return false;
  try {
    const parsed = new URL(u);
    return parsed.protocol === "http:" || parsed.protocol === "https:";
  } catch {
    return false;
  }
}

export function validateTriggerBody(raw: unknown): ValidateResult {
  if (!raw || typeof raw !== "object") return { ok: false, error: "Body must be a JSON object." };
  const b = raw as Record<string, unknown>;
  if (!(TRIGGER_ACTIONS as readonly string[]).includes(b.action as string)) {
    return { ok: false, error: `Unknown action. Supported: ${TRIGGER_ACTIONS.join(", ")}.` };
  }
  const action = b.action as TriggerAction;
  const payload = (b.payload ?? {}) as Record<string, unknown>;
  if (action === "show_qr" && !isValidUrl(payload.url)) {
    return { ok: false, error: "payload.url must be an http(s) URL ≤ 2048 chars." };
  }
  return { ok: true, action, payload };
}
```

- [ ] **Step 5: Run tests, verify pass**

Run: `npm run test -- api-scopes trigger-actions`
Expected: PASS.

- [ ] **Step 6: Commit**
```bash
git add lib/api-scopes.ts lib/api-scopes.test.ts lib/trigger-actions.ts lib/trigger-actions.test.ts
git commit -m "feat(credits): pure scope + trigger-action validation helpers

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Credit primitives (`lib/credits.ts`) — atomic guards

**Files:**
- Create: `lib/credits.ts`

**Interfaces:**
- Consumes: `db`, schema `creditBalance`/`creditLedger`, `id` from `lib/ids`.
- Produces:
  - `reserveCredit(args:{ organizationId; deviceId; action; commandId; cost }) → Promise<{ ok:true; availableAfter:number } | { ok:false; reason:"insufficient" }>`
  - `settleHold(args:{ organizationId; commandId; cost }) → Promise<void>`
  - `releaseHold(args:{ organizationId; commandId; cost }) → Promise<void>`
  - `grantCredits(args:{ organizationId; credits; kind:"grant"|"purchase"; note?; createdByUserId?; idempotencyKey? }) → Promise<{ applied:boolean }>`
  - `getBalance(organizationId) → Promise<{ available:number; held:number }>`

Ledger `credits` value convention (positive integers): `hold` = cost (moved available→held), `settle` = cost (held cleared; realized spend — **analytics counts settle rows**), `release` = cost (held→available), `grant`/`purchase` = credits added to available.

- [ ] **Step 1: Implement `lib/credits.ts`**
```ts
import { sql } from "drizzle-orm";
import { db } from "./db";
import { creditBalance, creditLedger } from "./db/schema";
import { id } from "./ids";

async function ledger(row: {
  organizationId: string; deviceId?: string | null; kind: "grant" | "purchase" | "hold" | "settle" | "release";
  credits: number; action?: string | null; commandId?: string | null; idempotencyKey?: string | null;
  balanceAfterAvailable?: number | null; note?: string | null; createdByUserId?: string | null;
}) {
  await db.insert(creditLedger).values({ id: id("cl"), ...row });
}

export async function getBalance(organizationId: string): Promise<{ available: number; held: number }> {
  const [b] = await db.select({ available: creditBalance.available, held: creditBalance.held })
    .from(creditBalance).where(sql`${creditBalance.organizationId} = ${organizationId}`).limit(1);
  return { available: b?.available ?? 0, held: b?.held ?? 0 };
}

/** Atomically move `cost` from available→held iff available >= cost. */
export async function reserveCredit(a: {
  organizationId: string; deviceId: string; action: string; commandId: string; cost: number;
}): Promise<{ ok: true; availableAfter: number } | { ok: false; reason: "insufficient" }> {
  const rows = await db.execute(sql`
    UPDATE credit_balance SET available = available - ${a.cost}, held = held + ${a.cost}, updated_at = now()
    WHERE organization_id = ${a.organizationId} AND available >= ${a.cost}
    RETURNING available`);
  const updated = rows.rows?.[0] as { available: number } | undefined;
  if (!updated) return { ok: false, reason: "insufficient" };
  await ledger({ organizationId: a.organizationId, deviceId: a.deviceId, kind: "hold", credits: a.cost,
    action: a.action, commandId: a.commandId, balanceAfterAvailable: updated.available });
  return { ok: true, availableAfter: updated.available };
}

/** Finalize a hold: clear it from `held` (credit truly spent). Idempotent via the held>=cost guard + caller's command-status lock. */
export async function settleHold(a: { organizationId: string; commandId: string; cost: number }): Promise<void> {
  const rows = await db.execute(sql`
    UPDATE credit_balance SET held = held - ${a.cost}, updated_at = now()
    WHERE organization_id = ${a.organizationId} AND held >= ${a.cost} RETURNING available`);
  const updated = rows.rows?.[0] as { available: number } | undefined;
  if (!updated) return; // nothing held → already settled/released
  await ledger({ organizationId: a.organizationId, kind: "settle", credits: a.cost,
    commandId: a.commandId, balanceAfterAvailable: updated.available });
}

/** Refund a hold: move it back held→available. */
export async function releaseHold(a: { organizationId: string; commandId: string; cost: number }): Promise<void> {
  const rows = await db.execute(sql`
    UPDATE credit_balance SET available = available + ${a.cost}, held = held - ${a.cost}, updated_at = now()
    WHERE organization_id = ${a.organizationId} AND held >= ${a.cost} RETURNING available`);
  const updated = rows.rows?.[0] as { available: number } | undefined;
  if (!updated) return;
  await ledger({ organizationId: a.organizationId, kind: "release", credits: a.cost,
    commandId: a.commandId, balanceAfterAvailable: updated.available });
}

/** Add credits. For `purchase`, the ledger row (unique on kind+idempotencyKey) is written FIRST; balance bumps only if it wrote. */
export async function grantCredits(a: {
  organizationId: string; credits: number; kind: "grant" | "purchase";
  note?: string; createdByUserId?: string; idempotencyKey?: string;
}): Promise<{ applied: boolean }> {
  if (a.kind === "purchase") {
    const ins = await db.execute(sql`
      INSERT INTO credit_ledger (id, organization_id, kind, credits, idempotency_key, note, created_at)
      VALUES (${id("cl")}, ${a.organizationId}, 'purchase', ${a.credits}, ${a.idempotencyKey ?? null}, ${a.note ?? null}, now())
      ON CONFLICT (kind, idempotency_key) DO NOTHING RETURNING id`);
    if (!ins.rows?.[0]) return { applied: false }; // replay
  }
  await db.execute(sql`
    INSERT INTO credit_balance (organization_id, available, held, updated_at)
    VALUES (${a.organizationId}, ${a.credits}, 0, now())
    ON CONFLICT (organization_id) DO UPDATE SET available = credit_balance.available + ${a.credits}, updated_at = now()`);
  if (a.kind === "grant") {
    await ledger({ organizationId: a.organizationId, kind: "grant", credits: a.credits,
      note: a.note ?? null, createdByUserId: a.createdByUserId ?? null });
  }
  return { applied: true };
}
```
> Note on `db.execute(sql\`...\`)`: this is Drizzle's raw-SQL escape hatch over `neon-http`; each call is one statement (atomic). Verify the exact `.rows` accessor shape against an existing `db.execute` usage in the repo (grep `db.execute(`); if the result shape differs, adjust the `rows.rows?.[0]` reads accordingly. Prefer Drizzle's typed `.update().set(...).where(...).returning()` if it can express `available - cost` with a `WHERE available >= cost` guard — `sql` raw is used here to guarantee the single-statement conditional decrement.

- [ ] **Step 2: Build**

Run: `npm run build`
Expected: compiles. Fix the `.rows` accessor if the repo's `db.execute` returns a different shape (grep an existing call first).

- [ ] **Step 3: Integration verification against the dev DB (throwaway script)**

Create `lib/db/_credits_probe.ts` operating ONLY on a dedicated test org id you create inline (prefix `org_credtest`), so it never touches real orgs:
```ts
import "./load-env";
import { db } from "../db";
import { organization, creditBalance, creditLedger } from "./schema";
import { eq } from "drizzle-orm";
import { reserveCredit, settleHold, releaseHold, grantCredits, getBalance } from "../credits";
async function main() {
  const org = "org_credtest_probe";
  await db.insert(organization).values({ id: org, name: "credtest", slug: "credtest-"+org, createdAt: new Date() }).onConflictDoNothing();
  // reset
  await db.delete(creditLedger).where(eq(creditLedger.organizationId, org));
  await db.delete(creditBalance).where(eq(creditBalance.organizationId, org));
  await grantCredits({ organizationId: org, credits: 2, kind: "grant" });
  console.log("after grant 2:", await getBalance(org)); // {available:2, held:0}
  const r1 = await reserveCredit({ organizationId: org, deviceId: "dev_x", action: "show_qr", commandId: "cmd_1", cost: 1 });
  console.log("reserve1:", r1, await getBalance(org)); // ok, {available:1, held:1}
  await settleHold({ organizationId: org, commandId: "cmd_1", cost: 1 });
  console.log("after settle:", await getBalance(org)); // {available:1, held:0}
  const r2 = await reserveCredit({ organizationId: org, deviceId: "dev_x", action: "show_qr", commandId: "cmd_2", cost: 1 });
  await releaseHold({ organizationId: org, commandId: "cmd_2", cost: 1 });
  console.log("after reserve+release:", await getBalance(org)); // {available:1, held:0}
  const r3 = await reserveCredit({ organizationId: org, deviceId: "dev_x", action: "show_qr", commandId: "cmd_3", cost: 2 });
  console.log("reserve over-balance:", r3); // {ok:false, reason:insufficient}
  const p1 = await grantCredits({ organizationId: org, credits: 5, kind: "purchase", idempotencyKey: "sess_1" });
  const p2 = await grantCredits({ organizationId: org, credits: 5, kind: "purchase", idempotencyKey: "sess_1" });
  console.log("purchase applied/replay:", p1, p2, await getBalance(org)); // applied:true, applied:false, available:6
  // cleanup
  await db.delete(creditLedger).where(eq(creditLedger.organizationId, org));
  await db.delete(creditBalance).where(eq(creditBalance.organizationId, org));
  await db.delete(organization).where(eq(organization.id, org));
  process.exit(0);
}
main();
```
Run `npx tsx lib/db/_credits_probe.ts`, confirm the printed assertions match the inline comments, then `rm lib/db/_credits_probe.ts`. (Adjust the `organization` insert columns to the real required NOT-NULL columns — check the schema.)

- [ ] **Step 4: Commit**
```bash
git add lib/credits.ts
git commit -m "feat(credits): atomic reserve/settle/release/grant primitives

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Trigger endpoint `POST /api/v1/devices/[deviceId]/trigger`

**Files:**
- Create: `app/api/v1/devices/[deviceId]/trigger/route.ts`

**Interfaces:**
- Consumes: `guardApiRequest`, `apiError`/`apiJson`, `hasScope`, `validateTriggerBody`/`creditCostForAction`, `reserveCredit`/`releaseHold`, `effectiveDeviceStatus`, schema `device`/`deviceCommand`/`apiKey`/`apiIdempotency`, `id`.
- Produces: the public endpoint; writes `deviceCommand{type:"trigger"}` + hold ledger + idempotency record.

- [ ] **Step 1: Implement the route**
```ts
import { and, eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { device as deviceTable, deviceCommand, apiKey as apiKeyTable, apiIdempotency } from "@/lib/db/schema";
import { guardApiRequest } from "@/lib/api/guard";
import { apiError, apiJson } from "@/lib/api/respond";
import { hasScope } from "@/lib/api-scopes";
import { validateTriggerBody, creditCostForAction } from "@/lib/trigger-actions";
import { reserveCredit, releaseHold } from "@/lib/credits";
import { effectiveDeviceStatus } from "@/lib/device-status";
import { id } from "@/lib/ids";

export const runtime = "nodejs";
const TTL_MS = 60_000;

export async function POST(req: Request, { params }: { params: Promise<{ deviceId: string }> }) {
  const guard = await guardApiRequest(req);
  if ("error" in guard) return guard.error;
  const { auth } = guard;

  // scope
  const [key] = await db.select({ scopes: apiKeyTable.scopes }).from(apiKeyTable).where(eq(apiKeyTable.id, auth.keyId)).limit(1);
  if (!hasScope(key?.scopes, "devices:trigger")) {
    return apiError("insufficient_scope", "API key lacks the devices:trigger scope.", 403);
  }

  // idempotency key required
  const idemKey = req.headers.get("idempotency-key")?.trim();
  if (!idemKey) return apiError("missing_idempotency_key", "Idempotency-Key header is required.", 400);
  const [prior] = await db.select().from(apiIdempotency)
    .where(and(eq(apiIdempotency.key, idemKey), eq(apiIdempotency.organizationId, auth.organizationId))).limit(1);
  if (prior) return apiJson(prior.responseBody, prior.responseStatus);

  // body
  let raw: unknown;
  try { raw = await req.json(); } catch { return apiError("invalid_request", "Malformed JSON body.", 422); }
  const v = validateTriggerBody(raw);
  if (!v.ok) return apiError("invalid_request", v.error, 422);

  // device ownership + eligibility
  const { deviceId } = await params;
  const [dev] = await db.select().from(deviceTable).where(eq(deviceTable.id, deviceId)).limit(1);
  if (!dev || dev.organizationId !== auth.organizationId) return apiError("device_not_found", "Device not found.", 404);
  if (effectiveDeviceStatus(dev.status, dev.lastSeenAt, new Date()) !== "online") {
    return apiError("device_offline", "Device is offline or paused.", 409);
  }

  // reserve
  const cost = creditCostForAction(v.action);
  const commandId = id("cmd");
  const reserved = await reserveCredit({ organizationId: auth.organizationId, deviceId, action: v.action, commandId, cost });
  if (!reserved.ok) return apiError("insufficient_credits", "Not enough credits.", 402);

  // enqueue
  try {
    await db.insert(deviceCommand).values({
      id: commandId, deviceId, organizationId: auth.organizationId, type: "trigger",
      status: "pending", action: v.action, payload: v.payload, expiresAt: new Date(Date.now() + TTL_MS),
    });
  } catch {
    await releaseHold({ organizationId: auth.organizationId, commandId, cost });
    return apiError("internal_error", "Could not enqueue the command.", 500);
  }

  const body = { id: commandId, status: "queued" as const };
  await db.insert(apiIdempotency).values({
    key: idemKey, organizationId: auth.organizationId, responseStatus: 202, responseBody: body, commandId,
  }).onConflictDoNothing();
  return apiJson(body, 202);
}
```
> `Date.now()` is allowed in app/server code (the no-`Date.now()` rule applies only to Workflow scripts).

- [ ] **Step 2: Build**

Run: `npm run build`
Expected: compiles.

- [ ] **Step 3: Manual integration check (dev DB + curl)**

Create a throwaway `lib/db/_seed_trigger_probe.ts` that: creates/loads a test org, sets its `credit_balance.available=3`, creates an online device (`status:"online"`, `lastSeenAt: new Date()`) in that org, and creates an API key with `scopes:["devices:trigger"]` returning the RAW key. Print `{ deviceId, rawKey }`. Then:
```bash
curl -s -X POST "http://localhost:3000/api/v1/devices/<deviceId>/trigger" \
  -H "Authorization: Bearer <rawKey>" -H "Idempotency-Key: test-1" \
  -H "Content-Type: application/json" -d '{"action":"show_qr","payload":{"url":"https://ex.co/r/abc"}}' -w "\n%{http_code}\n"
```
Expected `202 {"id":"cmd_…","status":"queued"}`. Re-run with the same `Idempotency-Key` → identical 202 (no second reserve). Try a key without the scope → 403; an offline device → 409; balance 0 → 402; bad body → 422; another org's device → 404. Verify the balance dropped by 1 and a `hold` ledger row exists. Delete the probe rows + script.

- [ ] **Step 4: Commit**
```bash
git add "app/api/v1/devices/[deviceId]/trigger/route.ts"
git commit -m "feat(credits): public device trigger endpoint (reserve + enqueue)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Settle on ack, deliver payload, release-on-expiry cron

**Files:**
- Modify: `app/api/device/commands/ack/route.ts`
- Modify: `app/api/device/commands/route.ts`
- Create: `app/api/cron/credit-holds/route.ts`
- Modify: `vercel.json`

**Interfaces:**
- Consumes: `settleHold`/`releaseHold`, `creditCostForAction`, schema.
- Produces: ack settles/releases the trigger hold; the poll response carries `action`+`payload`; the cron releases expired holds + cleans idempotency rows.

- [ ] **Step 1: Make ack settle/release via a guarded transition**

Replace the body of `app/api/device/commands/ack/route.ts`'s status update with a guarded transition that only the winner performs, then settle/release for trigger commands:
```ts
  const now = new Date();
  const nextStatus = body.ok ? "acked" : "failed";
  const [cmd] = await db.update(deviceCommand)
    .set({ status: nextStatus, ackedAt: now, result: body.result ?? null })
    .where(and(eq(deviceCommand.id, body.commandId), eq(deviceCommand.deviceId, device.id), eq(deviceCommand.status, "delivered")))
    .returning({ id: deviceCommand.id, type: deviceCommand.type, action: deviceCommand.action, organizationId: deviceCommand.organizationId });
  if (cmd && cmd.type === "trigger") {
    const cost = creditCostForAction((cmd.action ?? "show_qr") as "show_qr");
    if (body.ok) await settleHold({ organizationId: cmd.organizationId, commandId: cmd.id, cost });
    else await releaseHold({ organizationId: cmd.organizationId, commandId: cmd.id, cost });
  }
  return NextResponse.json({ ok: true });
```
(Add imports: `settleHold`, `releaseHold` from `@/lib/credits`; `creditCostForAction` from `@/lib/trigger-actions`. The `delivered`-only guard means a duplicate ack or an ack racing the cron's `expired` transition gets no row and moves nothing.)

- [ ] **Step 2: Deliver action+payload to the device**

In `app/api/device/commands/route.ts`, extend the `.returning(...)` to include action+payload:
```ts
    .returning({ id: deviceCommand.id, type: deviceCommand.type, action: deviceCommand.action, payload: deviceCommand.payload });
```
(The device ignores `action`/`payload` for non-trigger commands; nulls are harmless.)

- [ ] **Step 3: Release-on-expiry + idempotency-cleanup cron**

Create `app/api/cron/credit-holds/route.ts`:
```ts
import { NextResponse } from "next/server";
import { and, eq, inArray, lt, sql } from "drizzle-orm";
import { db } from "@/lib/db";
import { deviceCommand, apiIdempotency } from "@/lib/db/schema";
import { getEnv } from "@/lib/env";
import { releaseHold } from "@/lib/credits";
import { creditCostForAction } from "@/lib/trigger-actions";

export const runtime = "nodejs";

export async function GET(req: Request) {
  const secret = getEnv().CRON_SECRET;
  if (!secret) return NextResponse.json({ error: "CRON_SECRET not configured" }, { status: 503 });
  if (req.headers.get("authorization") !== `Bearer ${secret}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }
  const now = new Date();
  const expired = await db.select({ id: deviceCommand.id, organizationId: deviceCommand.organizationId, action: deviceCommand.action })
    .from(deviceCommand)
    .where(and(eq(deviceCommand.type, "trigger"), inArray(deviceCommand.status, ["pending", "delivered"]), lt(deviceCommand.expiresAt, now)));
  let released = 0;
  for (const c of expired) {
    const [won] = await db.update(deviceCommand).set({ status: "expired" })
      .where(and(eq(deviceCommand.id, c.id), inArray(deviceCommand.status, ["pending", "delivered"])))
      .returning({ id: deviceCommand.id });
    if (!won) continue; // lost the race to an ack
    await releaseHold({ organizationId: c.organizationId, commandId: c.id, cost: creditCostForAction((c.action ?? "show_qr") as "show_qr") });
    released++;
  }
  const del = await db.delete(apiIdempotency).where(lt(apiIdempotency.createdAt, new Date(now.getTime() - 24 * 60 * 60 * 1000))).returning({ key: apiIdempotency.key });
  return NextResponse.json({ ok: true, released, idempotencyPurged: del.length });
}
```

Add the cron to `vercel.json` (run every minute so 60s TTLs release promptly):
```json
    { "path": "/api/cron/credit-holds", "schedule": "* * * * *" }
```

- [ ] **Step 4: Build**

Run: `npm run build`
Expected: compiles.

- [ ] **Step 5: Integration check (extend the Task 4 probe)**

Using the same probe approach: trigger a command (202), then simulate the device ack by `curl -X POST /api/device/commands/ack` with the device key `{commandId, ok:true}` → verify the hold settled (`held` 0, a `settle` ledger row). Separately, insert a trigger command with `expiresAt` in the past + a reserved hold, hit `GET /api/cron/credit-holds` with the CRON_SECRET bearer → verify it released (status `expired`, `release` ledger row, `available` restored). Clean up.

- [ ] **Step 6: Commit**
```bash
git add "app/api/device/commands/ack/route.ts" "app/api/device/commands/route.ts" "app/api/cron/credit-holds/route.ts" vercel.json
git commit -m "feat(credits): settle on ack, deliver trigger payload, release-on-expiry cron

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: API-key scope picker

**Files:**
- Modify: `lib/actions/api-keys.ts`
- Modify: `components/api-key-create-dialog.tsx`

**Interfaces:**
- Consumes: `sanitizeScopes`, `DEFAULT_KEY_SCOPES`, `API_SCOPES`.
- Produces: keys created with chosen scopes.

- [ ] **Step 1: `createApiKey` reads + stores scopes**

In `lib/actions/api-keys.ts`, after the name validation, parse scopes from the form (multiple `scope` values) and default to `DEFAULT_KEY_SCOPES`:
```ts
import { sanitizeScopes, DEFAULT_KEY_SCOPES } from "@/lib/api-scopes";
// ...
  const chosen = sanitizeScopes(formData.getAll("scope"));
  const scopes = chosen.length ? chosen : DEFAULT_KEY_SCOPES;
```
Add `scopes` to the `db.insert(apiKeyTable).values({...})` and include `{ scopes }` in the audit metadata.

- [ ] **Step 2: Scope checkboxes in the dialog**

In `components/api-key-create-dialog.tsx`, render a checkbox per `API_SCOPES` entry (name `scope`, value the scope id), with `receipts:read` and `usage:read` checked by default and `devices:trigger` unchecked, with helper text "devices:trigger lets this key trigger devices and spend credits." Submit them in the existing FormData.
```tsx
import { API_SCOPES, DEFAULT_KEY_SCOPES } from "@/lib/api-scopes";
// inside the form, before the submit button:
<fieldset className="space-y-2">
  <legend className="text-sm font-medium">Permissions</legend>
  {API_SCOPES.map((s) => (
    <label key={s} className="flex items-center gap-2 text-sm">
      <input type="checkbox" name="scope" value={s} defaultChecked={DEFAULT_KEY_SCOPES.includes(s)} />
      <span className="font-mono">{s}</span>
    </label>
  ))}
  <p className="text-xs text-muted-foreground">devices:trigger lets this key trigger devices and spend credits.</p>
</fieldset>
```

- [ ] **Step 3: Build + manual**

Run: `npm run build`. Then manually create a key with `devices:trigger` checked and confirm (via the probe/DB) the row has the scope; create one without it and confirm it defaults to read-only.

- [ ] **Step 4: Commit**
```bash
git add lib/actions/api-keys.ts components/api-key-create-dialog.tsx
git commit -m "feat(credits): API-key scope picker (devices:trigger)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: Admin credit grant + balance/ledger view

**Files:**
- Create: `lib/actions/credits.ts`
- Modify: `app/(admin)/admin/customers/[tenantId]/page.tsx`
- Modify: `lib/data.ts` (a `getCreditLedger(orgId, limit)` reader for the view; `getBalance` reused from `lib/credits.ts`)

**Interfaces:**
- Produces: `grantCreditsAction(orgId, credits, note)` (platform-admin only); a balance + recent-ledger card on the admin customer page.

- [ ] **Step 1: `grantCreditsAction`**

`lib/actions/credits.ts`:
```ts
"use server";
import { requirePlatformAdmin } from "@/lib/session";
import { grantCredits } from "@/lib/credits";
import { recordAudit, AUDIT } from "@/lib/audit";
import { revalidatePath } from "next/cache";

export async function grantCreditsAction(formData: FormData): Promise<{ ok: boolean; error?: string }> {
  const ctx = await requirePlatformAdmin();
  const orgId = String(formData.get("organizationId") ?? "");
  const credits = Number(formData.get("credits") ?? 0);
  const note = String(formData.get("note") ?? "").trim() || undefined;
  if (!orgId || !Number.isInteger(credits) || credits <= 0 || credits > 1_000_000) {
    return { ok: false, error: "Enter a whole credit amount between 1 and 1,000,000." };
  }
  await grantCredits({ organizationId: orgId, credits, kind: "grant", note, createdByUserId: ctx.user.id });
  await recordAudit({ organizationId: orgId, actor: { type: "user", id: ctx.user.id, label: ctx.user.email },
    action: AUDIT.creditsGranted, metadata: { credits, note } });
  revalidatePath(`/admin/customers/${orgId}`);
  return { ok: true };
}
```
Add `creditsGranted: "credits.granted"` to the `AUDIT` map in `lib/audit.ts`.

- [ ] **Step 2: `getCreditLedger` reader in `lib/data.ts`**
```ts
export async function getCreditLedger(organizationId: string, limit = 50) {
  return db.select({
    id: creditLedger.id, kind: creditLedger.kind, credits: creditLedger.credits,
    deviceId: creditLedger.deviceId, action: creditLedger.action, note: creditLedger.note, createdAt: creditLedger.createdAt,
  }).from(creditLedger).where(eq(creditLedger.organizationId, organizationId))
    .orderBy(desc(creditLedger.createdAt)).limit(limit);
}
```
(Import `creditLedger`; `desc`/`eq` are already imported in data.ts.)

- [ ] **Step 3: Balance + grant card on the admin customer page**

In `app/(admin)/admin/customers/[tenantId]/page.tsx`, after the customer header card (~line 110), fetch `getBalance(tenantId)` + `getCreditLedger(tenantId)` and render a "Credits" card showing `available`/`held`, a small `grantCreditsAction` form (hidden `organizationId`, a `credits` number input, optional `note`), and a compact ledger table (kind, credits, device, time). Use the existing card/table components on the page.

- [ ] **Step 4: Build + manual**

Run: `npm run build`. As platform admin, open a customer, grant N credits → balance rises by N, a `grant` ledger row + audit entry appear.

- [ ] **Step 5: Commit**
```bash
git add lib/actions/credits.ts lib/audit.ts lib/data.ts "app/(admin)/admin/customers/[tenantId]/page.tsx"
git commit -m "feat(credits): platform-admin credit grant + balance/ledger view

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: Self-serve Stripe credit purchase

**Files:**
- Create: `lib/billing/credit-packs.ts`
- Modify: `lib/billing/stripe-billing.ts` (add `createCreditCheckout`)
- Modify: `app/api/stripe/webhook/route.ts` (add `checkout.session.completed`)
- Modify: `lib/env.ts` (add credit-pack price-id env)
- Modify: tenant billing UI (a "Buy credits" control)

**Interfaces:**
- Consumes: `grantCredits` (kind `purchase`), existing Stripe Checkout pattern.
- Produces: a one-time Checkout per pack; webhook grants credits idempotently.

- [ ] **Step 1: Credit-pack config**

`lib/env.ts`: add `STRIPE_CREDIT_PACK_PRICE_IDS: z.string().optional()` (comma list of `packId:priceId:credits`, e.g. `small:price_abc:100,large:price_def:1000`).

`lib/billing/credit-packs.ts`:
```ts
import { getEnv } from "@/lib/env";
export interface CreditPack { id: string; priceId: string; credits: number; }
export function creditPacks(): CreditPack[] {
  const raw = getEnv().STRIPE_CREDIT_PACK_PRICE_IDS ?? "";
  return raw.split(",").map((s) => s.trim()).filter(Boolean).map((s) => {
    const [id, priceId, credits] = s.split(":");
    return { id, priceId, credits: Number(credits) };
  }).filter((p) => p.id && p.priceId && Number.isFinite(p.credits) && p.credits > 0);
}
export function findPack(id: string): CreditPack | undefined { return creditPacks().find((p) => p.id === id); }
```

- [ ] **Step 2: `createCreditCheckout`**

In `lib/billing/stripe-billing.ts` (mirror `activateBilling`):
```ts
import { findPack } from "./credit-packs";
export async function createCreditCheckout(organizationId: string, packId: string): Promise<{ clientSecret: string }> {
  const s = requireStripe();
  const pack = findPack(packId);
  if (!pack) throw new Error("Unknown credit pack");
  const customerId = await ensureStripeCustomer(organizationId);
  const session = await s.checkout.sessions.create({
    mode: "payment", ui_mode: "elements", customer: customerId,
    line_items: [{ price: pack.priceId, quantity: 1 }],
    metadata: { organizationId, packId: pack.id, credits: String(pack.credits) },
    return_url: `${getEnv().BETTER_AUTH_URL}/tenant/billing`,
  });
  if (!session.client_secret) throw new Error("No client secret for credit checkout");
  return { clientSecret: session.client_secret };
}
```

- [ ] **Step 3: Webhook grants on `checkout.session.completed`**

In `app/api/stripe/webhook/route.ts`, add a case to the switch:
```ts
    case "checkout.session.completed": {
      const session = event.data.object as Stripe.Checkout.Session;
      if (session.mode === "payment" && session.payment_status === "paid" && session.metadata?.organizationId) {
        const credits = Number(session.metadata.credits);
        if (Number.isInteger(credits) && credits > 0) {
          const res = await grantCredits({ organizationId: session.metadata.organizationId, credits, kind: "purchase",
            idempotencyKey: session.id, note: `stripe pack ${session.metadata.packId ?? ""}` });
          if (res.applied) {
            await recordAudit({ organizationId: session.metadata.organizationId, actor: { type: "stripe" },
              action: AUDIT.creditsPurchased, metadata: { credits, sessionId: session.id } });
          }
        }
      }
      break;
    }
```
(Import `grantCredits` from `@/lib/credits`; add `creditsPurchased: "credits.purchased"` to `AUDIT`.)

- [ ] **Step 4: Tenant "Buy credits" control**

On the tenant billing page, render the `creditPacks()` as buy buttons that call a server action invoking `createCreditCheckout(org, packId)` and mount the Stripe Elements checkout the same way the existing `activateBilling` flow does (reuse that component pattern). Show the current `getBalance(org)` available credits.

- [ ] **Step 5: Build + manual (Stripe test mode)**

Run: `npm run build`. With Stripe test keys + a test credit-pack price configured, complete a Checkout in test mode and confirm the webhook grants the pack's credits exactly once (replay the event in the Stripe CLI → balance unchanged the second time).

- [ ] **Step 6: Commit**
```bash
git add lib/billing/credit-packs.ts lib/billing/stripe-billing.ts "app/api/stripe/webhook/route.ts" lib/env.ts lib/audit.ts <tenant billing files>
git commit -m "feat(credits): self-serve Stripe credit-pack purchase (idempotent webhook)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 9: Analytics — per-company / per-device usage

**Files:**
- Modify: `lib/data.ts` (queries) + a tiny pure mapper in `lib/credit-usage.ts` (+ test)
- Modify: `app/api/v1/usage/route.ts` (expose credit usage) and/or the tenant usage UI

**Interfaces:**
- Produces: `getCreditUsageByDevice(orgId, range)`, `getCreditUsageForOrg(orgId, range)`, `getCreditUsageAllOrgs(range)`; spend = sum of `settle` rows.

- [ ] **Step 1: Pure mapper + test (`lib/credit-usage.ts`)**

The DB returns raw grouped rows; a pure mapper shapes them. TDD the mapper:
`lib/credit-usage.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { rollupByDevice } from "./credit-usage";
describe("rollupByDevice", () => {
  it("sums settle credits per device and totals", () => {
    const r = rollupByDevice([
      { deviceId: "dev_a", credits: 1 }, { deviceId: "dev_a", credits: 1 }, { deviceId: "dev_b", credits: 1 },
    ]);
    expect(r.total).toBe(3);
    expect(r.byDevice).toEqual([{ deviceId: "dev_a", credits: 2, count: 2 }, { deviceId: "dev_b", credits: 1, count: 1 }]);
  });
});
```
`lib/credit-usage.ts`:
```ts
export function rollupByDevice(rows: { deviceId: string | null; credits: number }[]) {
  const m = new Map<string, { credits: number; count: number }>();
  let total = 0;
  for (const r of rows) {
    total += r.credits;
    const k = r.deviceId ?? "unknown";
    const cur = m.get(k) ?? { credits: 0, count: 0 };
    m.set(k, { credits: cur.credits + r.credits, count: cur.count + 1 });
  }
  return { total, byDevice: [...m].map(([deviceId, v]) => ({ deviceId, ...v })) };
}
```

- [ ] **Step 2: Queries in `lib/data.ts`**

Add (settle rows = realized spend; bound by a date range):
```ts
export async function getCreditUsageByDevice(organizationId: string, since: Date) {
  const rows = await db.select({ deviceId: creditLedger.deviceId, credits: creditLedger.credits })
    .from(creditLedger)
    .where(and(eq(creditLedger.organizationId, organizationId), eq(creditLedger.kind, "settle"), gte(creditLedger.createdAt, since)));
  return rollupByDevice(rows);
}
export async function getCreditUsageAllOrgs(since: Date) {
  return db.select({ organizationId: creditLedger.organizationId, credits: sql<number>`sum(${creditLedger.credits})::int`, count: sql<number>`count(*)::int` })
    .from(creditLedger).where(and(eq(creditLedger.kind, "settle"), gte(creditLedger.createdAt, since)))
    .groupBy(creditLedger.organizationId);
}
```
(Imports: `rollupByDevice`, `creditLedger`, `gte`, `sql` — add any missing.)

- [ ] **Step 3: Surface it**

Extend `GET /api/v1/usage` (guarded) to include a `credits` block (`getBalance` + this-period `getCreditUsageByDevice`). Add a per-device credit-usage panel to the tenant usage UI and a "credits by company" panel to the admin dashboard via `getCreditUsageAllOrgs`.

- [ ] **Step 4: Tests + build + manual**

Run: `npm run test -- credit-usage` (pure mapper green) and `npm run build`. Manually: after the Task 5 settle probe, call `/api/v1/usage` with a scoped key and confirm the credits block reflects the settled trigger; confirm the admin "by company" panel lists the test org.

- [ ] **Step 5: Commit**
```bash
git add lib/credit-usage.ts lib/credit-usage.test.ts lib/data.ts "app/api/v1/usage/route.ts" <ui files>
git commit -m "feat(credits): per-company/per-device usage analytics off the ledger

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Self-Review notes

- **Spec coverage:** balance+ledger+atomic guards (T1,T3); scopes + enforcement + backfill (T1,T2,T4,T6); trigger endpoint with idempotency/ownership/online/reserve (T4); ack→settle + payload delivery + TTL release cron (T5); admin grant (T7); Stripe purchase idempotent (T8); analytics off settle rows (T9); error table (T4); device only encodes URL (no fetch) — enforced by the firmware being out of scope (Spec B) and the payload being opaque data here.
- **Type consistency:** `reserveCredit/settleHold/releaseHold/grantCredits/getBalance`, `hasScope`, `validateTriggerBody/creditCostForAction`, `sanitizeScopes/DEFAULT_KEY_SCOPES`, `creditLedger.kind` values, and the `deviceCommand` status/`expired` transition are used identically across tasks.
- **Test reality:** pure logic is TDD'd (T2, T9 mapper); DB primitives/endpoints/cron are verified by build + throwaway dev-DB probes (T3,T4,T5) — matching this repo's pure-only vitest suite. Flag at review if a real test-DB harness is preferred instead.
- **Known follow-ups (not this plan):** Spec B firmware executes `show_qr` + the real ack; prod ops must register the Stripe webhook endpoint + create credit-pack Prices; the `* * * * *` cron cadence should be confirmed against the Vercel plan's cron limits.
