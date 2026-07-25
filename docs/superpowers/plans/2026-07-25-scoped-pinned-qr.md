# Scoped Pinned QR (device / store / tenant) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pinned QR becomes a layered setting — tenant-wide, per-store, per-device with `inherit | custom | none` modes — managed from a new `/tenant/pinned-qr` page and new org/store public API endpoints; devices keep receiving only a resolved effective URL (no firmware change).

**Architecture:** Pin columns are added at each level (`tenant_settings`, `store`, `device.pin_mode`). A pure resolver (`lib/pin-resolve.ts`) computes the effective pin (`device > store > tenant`) and a pure planner computes affected devices + credit charge for any scoped change. One shared mutation core (`lib/pin-service.ts applyScopedPinChange`) does charge → write → per-device fan-out (existing `pin` deviceCommand + MQTT) → audit, and is consumed by server actions AND API routes.

**Tech Stack:** Next.js 16 App Router, Drizzle/Neon (`neon-http`, no interactive transactions), Better Auth, vitest (pure lib tests only — this repo does not unit-test routes), shadcn radix-nova.

**Spec:** `docs/superpowers/specs/2026-07-25-scoped-pinned-qr-design.md` — read it first.

## Global Constraints

- Money rule: **paid ⇔ the change sets a URL** (`PUT {url}` at any scope); charge = number of devices whose effective URL changes, one `spendCredit` call, whole operation rejected on insufficient balance. Clears, mode changes (`none`/`inherit`), same-URL no-ops, membership-driven inheritance: **free**.
- Firmware contract unchanged: devices only ever see `deviceCommand {type:"pin", payload:{url}}` and config `pin.url` with the *effective* URL.
- `pin_mode = 'custom'` ⇔ `pinned_url IS NOT NULL` is enforced by writes; reads treat `custom` with null URL as no pin.
- Charge-first ordering; neon-http has no transactions — keep the existing documented crash posture (`lib/pin-service.ts` header comment).
- Money stays integer credits; RBAC via `canManageTenant` (`lib/roles.ts`); tenant pages return fragments (no re-padding, `PageHeader`/`PageSection` primitives).
- Migration hygiene: hand-strip drizzle-generated SQL to just this feature's statements (see "Drizzle snapshot drift" memory).
- Gates after every task: `npx tsc --noEmit` and `npm test` must pass. Commit per task. Do NOT deploy — leave the working tree for the user's local test + batch deploy.
- shadcn is style `radix-nova` — never add `base`-style components.
- This is Next.js 16 — check `node_modules/next/dist/docs/` before using an API you're unsure about; `params`/`searchParams` are Promises in pages/routes.

---

### Task 1: Schema + migration

**Files:**
- Modify: `lib/db/schema.ts` (tenantSettings ~line 164, store ~line 231, device ~line 293)
- Create: `drizzle/0036_scoped_pinned_qr.sql` (via `npm run db:generate`, then hand-edit)

**Interfaces:**
- Produces: `tenantSettings.pinnedUrl/pinnedAt`, `store.pinMode/pinnedUrl/pinnedAt`, `device.pinMode` Drizzle columns used by every later task.

- [ ] **Step 1: Add columns to `lib/db/schema.ts`**

In `tenantSettings` (after the `printerConfig`-related fields, before `qr*` style fields or at the end of the column list — keep neighbors' comment style):

```ts
  // Tenant-wide pinned QR: devices whose pin mode resolves to "inherit" all the
  // way up show this URL while idle. Null = no tenant pin.
  pinnedUrl: text("pinned_url"),
  pinnedAt: timestamp("pinned_at"),
```

In `store` (after `timezone`):

```ts
    // Store-level pinned QR. pinMode: "inherit" = follow the tenant pin,
    // "custom" = pinnedUrl below, "none" = suppress any pin for this store's
    // inheriting devices. custom ⇔ pinnedUrl set (enforced by write paths).
    pinMode: text("pin_mode", { enum: ["inherit", "custom", "none"] })
      .default("inherit")
      .notNull(),
    pinnedUrl: text("pinned_url"),
    pinnedAt: timestamp("pinned_at"),
```

In `device` (next to the existing `pinnedUrl`/`pinnedAt`, extend the existing comment):

```ts
    // Pin mode: "custom" = show pinnedUrl below, "none" = never show a pin
    // even if the store/tenant has one, "inherit" = resolve store → tenant.
    // custom ⇔ pinnedUrl set (enforced by write paths).
    pinMode: text("pin_mode", { enum: ["inherit", "custom", "none"] })
      .default("inherit")
      .notNull(),
```

- [ ] **Step 2: Generate + strip the migration**

Run: `npm run db:generate`
Then open the new `drizzle/0036_*.sql` and replace its content with EXACTLY (drop any spurious FK churn — see Global Constraints):

```sql
ALTER TABLE "tenant_settings" ADD COLUMN "pinned_url" text;--> statement-breakpoint
ALTER TABLE "tenant_settings" ADD COLUMN "pinned_at" timestamp;--> statement-breakpoint
ALTER TABLE "store" ADD COLUMN "pin_mode" text DEFAULT 'inherit' NOT NULL;--> statement-breakpoint
ALTER TABLE "store" ADD COLUMN "pinned_url" text;--> statement-breakpoint
ALTER TABLE "store" ADD COLUMN "pinned_at" timestamp;--> statement-breakpoint
ALTER TABLE "device" ADD COLUMN "pin_mode" text DEFAULT 'inherit' NOT NULL;--> statement-breakpoint
UPDATE "device" SET "pin_mode" = 'custom' WHERE "pinned_url" IS NOT NULL;
```

- [ ] **Step 3: Apply**

Run: `npm run db:migrate`
⚠️ `.env.local` targets the production Neon DB (known project posture). These statements are additive and default-safe; the backfill only stamps `custom` where a pin already exists. Expected: migration applies cleanly.

- [ ] **Step 4: Gates + commit**

Run: `npx tsc --noEmit && npm test` — expected clean.

```bash
git add lib/db/schema.ts drizzle/
git commit -m "feat(pin): schema for scoped pinned QR — tenant/store pin columns + device pin_mode"
```

---

### Task 2: `PinMode` type + PUT-body validation

**Files:**
- Modify: `lib/pin.ts`
- Test: `lib/pin.test.ts` (append)

**Interfaces:**
- Consumes: existing `validatePinBody` (unchanged).
- Produces: `type PinMode = "inherit" | "custom" | "none"`; `validatePinPutBody(raw, opts?): PinPutBodyResult` where `PinPutBodyResult = { ok: true; kind: "url"; url: string } | { ok: true; kind: "mode"; mode: "none" | "inherit" } | { ok: false; error: string }`. `opts.allowMode` defaults true; org route passes `{ allowMode: false }`.

- [ ] **Step 1: Write failing tests** (append to `lib/pin.test.ts`)

```ts
import { validatePinPutBody } from "./pin";

describe("validatePinPutBody", () => {
  it("accepts {url} and reuses url validation", () => {
    expect(validatePinPutBody({ url: "https://x.co/m" })).toEqual({
      ok: true, kind: "url", url: "https://x.co/m",
    });
    expect(validatePinPutBody({ url: "ftp://x.co" }).ok).toBe(false);
  });
  it("accepts {mode:'none'} and {mode:'inherit'}", () => {
    expect(validatePinPutBody({ mode: "none" })).toEqual({ ok: true, kind: "mode", mode: "none" });
    expect(validatePinPutBody({ mode: "inherit" })).toEqual({ ok: true, kind: "mode", mode: "inherit" });
  });
  it("rejects mode:'custom' (custom is expressed by sending a url)", () => {
    expect(validatePinPutBody({ mode: "custom" }).ok).toBe(false);
  });
  it("rejects both url and mode together", () => {
    expect(validatePinPutBody({ url: "https://x.co", mode: "none" }).ok).toBe(false);
  });
  it("rejects mode when allowMode is false (org scope)", () => {
    expect(validatePinPutBody({ mode: "none" }, { allowMode: false }).ok).toBe(false);
  });
  it("rejects empty objects and non-objects", () => {
    expect(validatePinPutBody({}).ok).toBe(false);
    expect(validatePinPutBody(null).ok).toBe(false);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test -- lib/pin.test.ts`
Expected: FAIL — `validatePinPutBody` is not exported.

- [ ] **Step 3: Implement** (append to `lib/pin.ts`)

```ts
/** Pin mode shared by device and store levels. */
export type PinMode = "inherit" | "custom" | "none";

export type PinPutBodyResult =
  | { ok: true; kind: "url"; url: string }
  | { ok: true; kind: "mode"; mode: "none" | "inherit" }
  | { ok: false; error: string };

/**
 * PUT body for the scoped pin endpoints: {url} sets a custom pin (paid),
 * {mode:"none"|"inherit"} switches mode (free). "custom" mode is only ever
 * expressed by sending a url. Org scope has no modes → allowMode: false.
 */
export function validatePinPutBody(
  raw: unknown,
  opts: { allowMode?: boolean } = {},
): PinPutBodyResult {
  const allowMode = opts.allowMode ?? true;
  if (typeof raw !== "object" || raw === null) {
    return { ok: false, error: "Body must be a JSON object." };
  }
  const { url, mode } = raw as { url?: unknown; mode?: unknown };
  if (url !== undefined && mode !== undefined) {
    return { ok: false, error: "Send either `url` or `mode`, not both." };
  }
  if (mode !== undefined) {
    if (!allowMode) return { ok: false, error: "`mode` is not supported at this scope; send `url`." };
    if (mode !== "none" && mode !== "inherit") {
      return { ok: false, error: "`mode` must be \"none\" or \"inherit\" (custom = send `url`)." };
    }
    return { ok: true, kind: "mode", mode };
  }
  const v = validatePinBody(raw);
  if (!v.ok) return v;
  return { ok: true, kind: "url", url: v.url };
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test -- lib/pin.test.ts` — expected PASS. Then `npx tsc --noEmit`.

- [ ] **Step 5: Commit**

```bash
git add lib/pin.ts lib/pin.test.ts
git commit -m "feat(pin): PinMode type + validatePinPutBody for scoped pin endpoints"
```

---

### Task 3: Effective-pin resolver (pure)

**Files:**
- Create: `lib/pin-resolve.ts`
- Test: `lib/pin-resolve.test.ts`

**Interfaces:**
- Consumes: `PinMode` from `lib/pin.ts`.
- Produces (used by config route, data layer, planner, UI):

```ts
export interface PinLevel { pinMode: PinMode; pinnedUrl: string | null }
export interface EffectivePin { url: string | null; source: "device" | "store" | "tenant" | null }
export function resolveEffectivePin(a: {
  device: PinLevel;
  store: PinLevel | null;            // null = pool device (no store)
  tenant: { pinnedUrl: string | null };
}): EffectivePin
```

- [ ] **Step 1: Write failing tests** (`lib/pin-resolve.test.ts`)

```ts
import { describe, it, expect } from "vitest";
import { resolveEffectivePin } from "./pin-resolve";

const T = { pinnedUrl: "https://tenant.example/t" };
const noTenant = { pinnedUrl: null };
const inherit = { pinMode: "inherit" as const, pinnedUrl: null };

describe("resolveEffectivePin", () => {
  it("device custom wins over everything", () => {
    expect(
      resolveEffectivePin({
        device: { pinMode: "custom", pinnedUrl: "https://d.example" },
        store: { pinMode: "custom", pinnedUrl: "https://s.example" },
        tenant: T,
      }),
    ).toEqual({ url: "https://d.example", source: "device" });
  });
  it("device none suppresses store and tenant pins", () => {
    expect(
      resolveEffectivePin({
        device: { pinMode: "none", pinnedUrl: null },
        store: { pinMode: "custom", pinnedUrl: "https://s.example" },
        tenant: T,
      }),
    ).toEqual({ url: null, source: null });
  });
  it("inheriting device gets the store custom pin", () => {
    expect(
      resolveEffectivePin({
        device: inherit,
        store: { pinMode: "custom", pinnedUrl: "https://s.example" },
        tenant: T,
      }),
    ).toEqual({ url: "https://s.example", source: "store" });
  });
  it("store none suppresses the tenant pin", () => {
    expect(
      resolveEffectivePin({ device: inherit, store: { pinMode: "none", pinnedUrl: null }, tenant: T }),
    ).toEqual({ url: null, source: null });
  });
  it("full inherit chain reaches the tenant pin", () => {
    expect(resolveEffectivePin({ device: inherit, store: inherit, tenant: T })).toEqual({
      url: "https://tenant.example/t",
      source: "tenant",
    });
  });
  it("pool device (store null) inherits the tenant pin directly", () => {
    expect(resolveEffectivePin({ device: inherit, store: null, tenant: T })).toEqual({
      url: "https://tenant.example/t",
      source: "tenant",
    });
  });
  it("no pin anywhere resolves to null/null", () => {
    expect(resolveEffectivePin({ device: inherit, store: inherit, tenant: noTenant })).toEqual({
      url: null,
      source: null,
    });
  });
  it("tolerates custom with a null url (treated as no pin at that level)", () => {
    expect(
      resolveEffectivePin({ device: { pinMode: "custom", pinnedUrl: null }, store: inherit, tenant: T }),
    ).toEqual({ url: "https://tenant.example/t", source: "tenant" });
    expect(
      resolveEffectivePin({ device: inherit, store: { pinMode: "custom", pinnedUrl: null }, tenant: T }),
    ).toEqual({ url: "https://tenant.example/t", source: "tenant" });
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test -- lib/pin-resolve.test.ts` — expected FAIL (module not found).

- [ ] **Step 3: Implement** (`lib/pin-resolve.ts`)

```ts
// lib/pin-resolve.ts
// Pure effective-pin resolution + change planning for scoped pinned QR.
// The precedence is device > store > tenant; "none" stops the chain, "custom"
// answers it, "inherit" delegates upward. Devices only ever see the resolved
// URL (spec: docs/superpowers/specs/2026-07-25-scoped-pinned-qr-design.md).

import type { PinMode } from "@/lib/pin";

export interface PinLevel {
  pinMode: PinMode;
  pinnedUrl: string | null;
}

export interface EffectivePin {
  url: string | null;
  source: "device" | "store" | "tenant" | null;
}

export function resolveEffectivePin(a: {
  device: PinLevel;
  store: PinLevel | null; // null = pool device
  tenant: { pinnedUrl: string | null };
}): EffectivePin {
  // "custom" with a null URL is tolerated data drift → treated as inherit.
  if (a.device.pinMode === "custom" && a.device.pinnedUrl !== null) {
    return { url: a.device.pinnedUrl, source: "device" };
  }
  if (a.device.pinMode === "none") return { url: null, source: null };
  if (a.store) {
    if (a.store.pinMode === "custom" && a.store.pinnedUrl !== null) {
      return { url: a.store.pinnedUrl, source: "store" };
    }
    if (a.store.pinMode === "none") return { url: null, source: null };
  }
  return a.tenant.pinnedUrl !== null
    ? { url: a.tenant.pinnedUrl, source: "tenant" }
    : { url: null, source: null };
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test -- lib/pin-resolve.test.ts` — expected PASS. Then `npx tsc --noEmit`.

- [ ] **Step 5: Commit**

```bash
git add lib/pin-resolve.ts lib/pin-resolve.test.ts
git commit -m "feat(pin): pure effective-pin resolver (device > store > tenant)"
```

---

### Task 4: Change planner (pure) — affected devices + charge

**Files:**
- Modify: `lib/pin-resolve.ts` (append)
- Test: `lib/pin-resolve.test.ts` (append)

**Interfaces:**
- Produces (used by `applyScopedPinChange` in Task 5 and cost previews in Task 8):

```ts
export type ScopedPinChange =
  | { scope: "org"; url: string | null }                                  // null = clear
  | { scope: "store"; storeId: string; mode: PinMode; url: string | null } // custom ⇔ url set
  | { scope: "device"; deviceId: string; mode: PinMode; url: string | null };
export interface DevicePinRow extends PinLevel { id: string; storeId: string | null }
export interface StorePinRow extends PinLevel { id: string }
export function planScopedPinChange(a: {
  devices: DevicePinRow[];       // ALL the org's devices
  stores: StorePinRow[];         // ALL the org's stores
  tenantPinnedUrl: string | null;
  change: ScopedPinChange;
}): { affected: { deviceId: string; newUrl: string | null }[]; chargedCount: number }
```

`chargedCount = change sets a URL ? affected.length : 0` (Global Constraints money rule).

- [ ] **Step 1: Write failing tests** (append to `lib/pin-resolve.test.ts`)

```ts
import { planScopedPinChange, type DevicePinRow, type StorePinRow } from "./pin-resolve";

describe("planScopedPinChange", () => {
  const stores: StorePinRow[] = [
    { id: "s1", pinMode: "inherit", pinnedUrl: null },
    { id: "s2", pinMode: "custom", pinnedUrl: "https://s2.example" },
    { id: "s3", pinMode: "none", pinnedUrl: null },
  ];
  const devices: DevicePinRow[] = [
    { id: "d1", storeId: "s1", pinMode: "inherit", pinnedUrl: null },   // follows tenant
    { id: "d2", storeId: "s1", pinMode: "custom", pinnedUrl: "https://d2.example" },
    { id: "d3", storeId: "s2", pinMode: "inherit", pinnedUrl: null },   // follows s2
    { id: "d4", storeId: "s3", pinMode: "inherit", pinnedUrl: null },   // suppressed by s3
    { id: "d5", storeId: null, pinMode: "inherit", pinnedUrl: null },   // pool → tenant
    { id: "d6", storeId: "s1", pinMode: "none", pinnedUrl: null },      // opted out
  ];

  it("org set reaches only devices that resolve to tenant, and charges them", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: null,
      change: { scope: "org", url: "https://t.example" },
    });
    expect(r.affected).toEqual([
      { deviceId: "d1", newUrl: "https://t.example" },
      { deviceId: "d5", newUrl: "https://t.example" },
    ]);
    expect(r.chargedCount).toBe(2);
  });

  it("org clear is free but still fans out to followers", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: "https://t.example",
      change: { scope: "org", url: null },
    });
    expect(r.affected).toEqual([
      { deviceId: "d1", newUrl: null },
      { deviceId: "d5", newUrl: null },
    ]);
    expect(r.chargedCount).toBe(0);
  });

  it("store url set affects only that store's inheriting devices", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: null,
      change: { scope: "store", storeId: "s1", mode: "custom", url: "https://new.example" },
    });
    expect(r.affected).toEqual([{ deviceId: "d1", newUrl: "https://new.example" }]);
    expect(r.chargedCount).toBe(1);
  });

  it("same-URL devices are not affected (free no-op per device)", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: "https://s2.example",
      change: { scope: "store", storeId: "s2", mode: "inherit", url: null },
    });
    // d3's effective stays https://s2.example (now from tenant) → not affected.
    expect(r.affected).toEqual([]);
    expect(r.chargedCount).toBe(0);
  });

  it("store mode inherit is free even when devices gain a pin", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: "https://t.example",
      change: { scope: "store", storeId: "s3", mode: "inherit", url: null },
    });
    expect(r.affected).toEqual([{ deviceId: "d4", newUrl: "https://t.example" }]);
    expect(r.chargedCount).toBe(0);
  });

  it("device url set affects exactly that device", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: null,
      change: { scope: "device", deviceId: "d1", mode: "custom", url: "https://d1.example" },
    });
    expect(r.affected).toEqual([{ deviceId: "d1", newUrl: "https://d1.example" }]);
    expect(r.chargedCount).toBe(1);
  });

  it("device mode none is free and clears its effective pin", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: "https://t.example",
      change: { scope: "device", deviceId: "d1", mode: "none", url: null },
    });
    expect(r.affected).toEqual([{ deviceId: "d1", newUrl: null }]);
    expect(r.chargedCount).toBe(0);
  });

  it("device DELETE→inherit falls back through the chain, free", () => {
    const r = planScopedPinChange({
      devices, stores, tenantPinnedUrl: null,
      change: { scope: "device", deviceId: "d2", mode: "inherit", url: null },
    });
    // d2 was custom https://d2.example; s1 inherits and there is no tenant pin.
    expect(r.affected).toEqual([{ deviceId: "d2", newUrl: null }]);
    expect(r.chargedCount).toBe(0);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npm test -- lib/pin-resolve.test.ts` — expected FAIL (`planScopedPinChange` not exported).

- [ ] **Step 3: Implement** (append to `lib/pin-resolve.ts`)

```ts
export type ScopedPinChange =
  | { scope: "org"; url: string | null }
  | { scope: "store"; storeId: string; mode: PinMode; url: string | null }
  | { scope: "device"; deviceId: string; mode: PinMode; url: string | null };

export interface DevicePinRow extends PinLevel {
  id: string;
  storeId: string | null;
}
export interface StorePinRow extends PinLevel {
  id: string;
}

/** True when the change is a paid URL set (Global money rule: paid ⇔ sets a URL). */
export function changeSetsUrl(change: ScopedPinChange): boolean {
  return change.url !== null;
}

/**
 * Compute which devices' effective pin URL a scoped change would alter, and
 * how many credits it costs. Pure: callers load rows, we only do math.
 */
export function planScopedPinChange(a: {
  devices: DevicePinRow[];
  stores: StorePinRow[];
  tenantPinnedUrl: string | null;
  change: ScopedPinChange;
}): { affected: { deviceId: string; newUrl: string | null }[]; chargedCount: number } {
  const storeById = new Map(a.stores.map((s) => [s.id, s]));

  // Apply the change to a copy of the three levels.
  const nextTenantUrl = a.change.scope === "org" ? a.change.url : a.tenantPinnedUrl;
  const nextStore = (s: StorePinRow | null): PinLevel | null => {
    if (!s) return null;
    if (a.change.scope === "store" && a.change.storeId === s.id) {
      return { pinMode: a.change.mode, pinnedUrl: a.change.url };
    }
    return s;
  };
  const nextDevice = (d: DevicePinRow): PinLevel => {
    if (a.change.scope === "device" && a.change.deviceId === d.id) {
      return { pinMode: a.change.mode, pinnedUrl: a.change.url };
    }
    return d;
  };

  const affected: { deviceId: string; newUrl: string | null }[] = [];
  for (const d of a.devices) {
    const store = d.storeId ? (storeById.get(d.storeId) ?? null) : null;
    const before = resolveEffectivePin({
      device: d,
      store,
      tenant: { pinnedUrl: a.tenantPinnedUrl },
    });
    const after = resolveEffectivePin({
      device: nextDevice(d),
      store: nextStore(store),
      tenant: { pinnedUrl: nextTenantUrl },
    });
    if (before.url !== after.url) affected.push({ deviceId: d.id, newUrl: after.url });
  }
  return { affected, chargedCount: changeSetsUrl(a.change) ? affected.length : 0 };
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npm test -- lib/pin-resolve.test.ts` — expected PASS. Then `npx tsc --noEmit`.

- [ ] **Step 5: Commit**

```bash
git add lib/pin-resolve.ts lib/pin-resolve.test.ts
git commit -m "feat(pin): pure scoped-change planner — affected devices + credit charge"
```

---

### Task 5: Mutation core — `applyScopedPinChange` + `pushEffectivePin`

**Files:**
- Modify: `lib/pin-service.ts` (rewrite most of it), `lib/audit.ts:35-36` area, `lib/audit-labels.ts:26-27` area

**Interfaces:**
- Consumes: `planScopedPinChange`, `changeSetsUrl`, `resolveEffectivePin`, `ScopedPinChange` (Task 4); `spendCredit` (`lib/credits.ts`, takes `cost`); `publishCommand` (`lib/mqtt.ts`); `recordAudit`/`AUDIT` (`lib/audit.ts`).
- Prerequisite tweak: widen `spendCredit`'s `deviceId` param (`lib/credits.ts:151`) from `string` to `string | null` — org/store-scoped charges have no single device. The `credit_ledger.device_id` column is already nullable (`lib/db/schema.ts:423`); if the internal `ledger()` helper types `deviceId` as `string`, widen it to `string | null` too. Existing callers pass strings and keep compiling.
- Produces:

```ts
export type ScopedPinResult =
  | { ok: true; noop: boolean; affectedDevices: number; creditsCharged: number; pinnedAt: Date | null }
  | { ok: false; reason: "insufficient_credits"; required: number };
export async function applyScopedPinChange(a: {
  organizationId: string;
  change: ScopedPinChange;
  actor: AuditActor;
  via: "api" | "ui";
  createdByUserId?: string | null;
}): Promise<ScopedPinResult>
export async function pushEffectivePin(organizationId: string, deviceIds: string[]): Promise<void>
// Back-compat thin wrappers (existing callers keep compiling; semantics per spec):
export async function setDevicePin(a: {...existing signature...}): Promise<SetPinResult>   // → applyScopedPinChange device/custom
export async function clearDevicePin(a: {...existing signature...}): Promise<void>          // → device/inherit (NEW semantics: falls back to store/tenant)
```

- [ ] **Step 1: Add audit constants + labels**

`lib/audit.ts` — insert after `devicePinCleared` (line 36):

```ts
  devicePinModeNone: "device.pin_mode_none",
  orgPinSet: "org.pin_set",
  orgPinCleared: "org.pin_cleared",
  storePinSet: "store.pin_set",
  storePinCleared: "store.pin_cleared",
  storePinModeNone: "store.pin_mode_none",
```

`lib/audit-labels.ts` — next to the existing pin labels (lines 26-27):

```ts
  "device.pin_mode_none": "Device pin disabled",
  "org.pin_set": "Tenant-wide pinned QR set",
  "org.pin_cleared": "Tenant-wide pinned QR removed",
  "store.pin_set": "Store pinned QR set",
  "store.pin_cleared": "Store pinned QR removed",
  "store.pin_mode_none": "Store pin disabled",
```

Audit mapping used below: URL set → `*PinSet`; clear/DELETE/`inherit` → `*PinCleared` (device: existing `devicePinCleared`); mode `none` → `*PinModeNone`. Metadata always includes `via`, `affectedDevices`, `creditsCharged`, plus `url` on sets and `mode` on mode changes.

- [ ] **Step 2: Rewrite `lib/pin-service.ts`**

Keep the file header comment (update wording to "scoped"), `PIN_COST`, and `enqueuePinCommand` exactly as they are. Replace `setDevicePin`/`clearDevicePin` bodies and add the core:

```ts
import { eq, inArray } from "drizzle-orm";
import { db } from "@/lib/db";
import {
  device as deviceTable,
  store as storeTable,
  tenantSettings,
  deviceCommand,
} from "@/lib/db/schema";
import { spendCredit } from "@/lib/credits";
import { id } from "@/lib/ids";
import { publishCommand } from "@/lib/mqtt";
import { recordAudit, AUDIT, type AuditActor } from "@/lib/audit";
import {
  planScopedPinChange,
  resolveEffectivePin,
  changeSetsUrl,
  type ScopedPinChange,
  type DevicePinRow,
  type StorePinRow,
} from "@/lib/pin-resolve";
import type { PinMode } from "@/lib/pin";

export type ScopedPinResult =
  | { ok: true; noop: boolean; affectedDevices: number; creditsCharged: number; pinnedAt: Date | null }
  | { ok: false; reason: "insufficient_credits"; required: number };

async function loadPinWorld(organizationId: string): Promise<{
  devices: DevicePinRow[];
  stores: StorePinRow[];
  tenantPinnedUrl: string | null;
}> {
  const [devices, stores, [ts]] = await Promise.all([
    db
      .select({
        id: deviceTable.id,
        storeId: deviceTable.storeId,
        pinMode: deviceTable.pinMode,
        pinnedUrl: deviceTable.pinnedUrl,
      })
      .from(deviceTable)
      .where(eq(deviceTable.organizationId, organizationId)),
    db
      .select({ id: storeTable.id, pinMode: storeTable.pinMode, pinnedUrl: storeTable.pinnedUrl })
      .from(storeTable)
      .where(eq(storeTable.organizationId, organizationId)),
    db
      .select({ pinnedUrl: tenantSettings.pinnedUrl })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, organizationId)),
  ]);
  return { devices, stores, tenantPinnedUrl: ts?.pinnedUrl ?? null };
}

/** Is the change a no-op against the level's current stored state? */
function isLevelNoop(
  change: ScopedPinChange,
  world: { stores: StorePinRow[]; devices: DevicePinRow[]; tenantPinnedUrl: string | null },
): boolean {
  if (change.scope === "org") return world.tenantPinnedUrl === change.url;
  if (change.scope === "store") {
    const s = world.stores.find((x) => x.id === change.storeId);
    return !!s && s.pinMode === change.mode && s.pinnedUrl === change.url;
  }
  const d = world.devices.find((x) => x.id === change.deviceId);
  return !!d && d.pinMode === change.mode && d.pinnedUrl === change.url;
}

export async function applyScopedPinChange(a: {
  organizationId: string;
  change: ScopedPinChange;
  actor: AuditActor;
  via: "api" | "ui";
  createdByUserId?: string | null;
}): Promise<ScopedPinResult> {
  const world = await loadPinWorld(a.organizationId);
  if (isLevelNoop(a.change, world)) {
    return { ok: true, noop: true, affectedDevices: 0, creditsCharged: 0, pinnedAt: null };
  }
  const plan = planScopedPinChange({ ...world, change: a.change });

  // Paid ⇔ the change sets a URL (charge-first; see file header for the
  // crash posture on neon-http's lack of transactions).
  if (plan.chargedCount > 0) {
    const spent = await spendCredit({
      organizationId: a.organizationId,
      deviceId: a.change.scope === "device" ? a.change.deviceId : null,
      action: "pin_change",
      cost: plan.chargedCount * PIN_COST,
      createdByUserId: a.createdByUserId ?? null,
    });
    if (!spent.ok) {
      return { ok: false, reason: "insufficient_credits", required: plan.chargedCount * PIN_COST };
    }
  }

  const pinnedAt = a.change.url !== null ? new Date() : null;
  if (a.change.scope === "org") {
    await db
      .update(tenantSettings)
      .set({ pinnedUrl: a.change.url, pinnedAt })
      .where(eq(tenantSettings.organizationId, a.organizationId));
  } else if (a.change.scope === "store") {
    await db
      .update(storeTable)
      .set({ pinMode: a.change.mode, pinnedUrl: a.change.url, pinnedAt })
      .where(eq(storeTable.id, a.change.storeId));
  } else {
    await db
      .update(deviceTable)
      .set({ pinMode: a.change.mode, pinnedUrl: a.change.url, pinnedAt })
      .where(eq(deviceTable.id, a.change.deviceId));
  }

  for (const dev of plan.affected) {
    await enqueuePinCommand({
      organizationId: a.organizationId,
      deviceId: dev.deviceId,
      url: dev.newUrl,
    });
  }

  const set = changeSetsUrl(a.change);
  const modeNone = !set && a.change.scope !== "org" && a.change.mode === "none";
  const action =
    a.change.scope === "org"
      ? set ? AUDIT.orgPinSet : AUDIT.orgPinCleared
      : a.change.scope === "store"
        ? set ? AUDIT.storePinSet : modeNone ? AUDIT.storePinModeNone : AUDIT.storePinCleared
        : set ? AUDIT.devicePinSet : modeNone ? AUDIT.devicePinModeNone : AUDIT.devicePinCleared;
  const target =
    a.change.scope === "org"
      ? { type: "organization", id: a.organizationId }
      : a.change.scope === "store"
        ? { type: "store", id: a.change.storeId }
        : { type: "device", id: a.change.deviceId };
  await recordAudit({
    organizationId: a.organizationId,
    actor: a.actor,
    action,
    target,
    metadata: {
      via: a.via,
      ...(set ? { url: a.change.url } : {}),
      ...(a.change.scope !== "org" && !set ? { mode: a.change.mode } : {}),
      affectedDevices: plan.affected.length,
      creditsCharged: plan.chargedCount * PIN_COST,
    },
  });
  return {
    ok: true,
    noop: false,
    affectedDevices: plan.affected.length,
    creditsCharged: plan.chargedCount * PIN_COST,
    pinnedAt,
  };
}

/**
 * Re-deliver the CURRENT effective pin to the given devices (free). Used after
 * membership changes (claim, move, store deletion) — idempotent on-device.
 */
export async function pushEffectivePin(organizationId: string, deviceIds: string[]): Promise<void> {
  if (deviceIds.length === 0) return;
  const world = await loadPinWorld(organizationId);
  const storeById = new Map(world.stores.map((s) => [s.id, s]));
  const rows = await db
    .select({ id: deviceTable.id, storeId: deviceTable.storeId, pinMode: deviceTable.pinMode, pinnedUrl: deviceTable.pinnedUrl })
    .from(deviceTable)
    .where(inArray(deviceTable.id, deviceIds));
  for (const d of rows) {
    const eff = resolveEffectivePin({
      device: d,
      store: d.storeId ? (storeById.get(d.storeId) ?? null) : null,
      tenant: { pinnedUrl: world.tenantPinnedUrl },
    });
    await enqueuePinCommand({ organizationId, deviceId: d.id, url: eff.url });
  }
}
```

Then reimplement the two existing exports as thin wrappers (keep their exact signatures and `SetPinResult` so Task-9/12 callers compile until they're reworked):

```ts
export async function setDevicePin(a: {
  organizationId: string;
  device: { id: string; pinnedUrl: string | null; pinnedAt: Date | null };
  url: string;
  actor: AuditActor;
  via: "api" | "ui";
  createdByUserId?: string | null;
}): Promise<SetPinResult> {
  const res = await applyScopedPinChange({
    organizationId: a.organizationId,
    change: { scope: "device", deviceId: a.device.id, mode: "custom", url: a.url },
    actor: a.actor,
    via: a.via,
    createdByUserId: a.createdByUserId,
  });
  if (!res.ok) return { ok: false, reason: "insufficient_credits" };
  return { ok: true, noop: res.noop, pinnedAt: res.pinnedAt ?? a.device.pinnedAt ?? new Date() };
}

/** NOTE: semantics change per spec — "clear" now means → inherit (falls back
 *  to a store/tenant pin when one exists), not "guaranteed blank". */
export async function clearDevicePin(a: {
  organizationId: string;
  device: { id: string; pinnedUrl: string | null };
  actor: AuditActor;
  via: "api" | "ui";
}): Promise<void> {
  await applyScopedPinChange({
    organizationId: a.organizationId,
    change: { scope: "device", deviceId: a.device.id, mode: "inherit", url: null },
    actor: a.actor,
    via: a.via,
  });
}
```

Delete the now-unused direct bodies (the old `setDevicePin`/`clearDevicePin` logic). Note: the old same-URL no-op and idempotent-clear checks now live in `isLevelNoop` (a device with `pinMode:"custom"` and the same URL → noop; clearing an already-inherit device → noop).

- [ ] **Step 3: Gates**

Run: `npx tsc --noEmit && npm test` — expected clean (existing callers use the wrappers).

- [ ] **Step 4: Commit**

```bash
git add lib/pin-service.ts lib/audit.ts lib/audit-labels.ts lib/credits.ts
git commit -m "feat(pin): scoped mutation core — applyScopedPinChange + pushEffectivePin + audit actions"
```

---

### Task 6: Config route serves the effective pin

**Files:**
- Modify: `app/api/device/config/route.ts:19-23`

**Interfaces:**
- Consumes: `resolveEffectivePin` (Task 3); `getDeviceConfig(organizationId, ifNoneMatch, pin?)` (`lib/data.ts:1235`, unchanged — its `pin` param already flows to `payload.pin`).

- [ ] **Step 1: Resolve before calling `getDeviceConfig`**

Replace lines 19-23 (`const { version, notModified, payload } = await getDeviceConfig(...)`) with:

```ts
  // Effective pin: device > store > tenant (lib/pin-resolve.ts). The device
  // only ever sees the resolved URL. pinnedUrl/pinMode/storeId come from the
  // authenticated device row.
  const [storeRow, [ts]] = await Promise.all([
    device.storeId
      ? db
          .select({ pinMode: storeTable.pinMode, pinnedUrl: storeTable.pinnedUrl })
          .from(storeTable)
          .where(eq(storeTable.id, device.storeId))
          .limit(1)
          .then((r) => r[0] ?? null)
      : Promise.resolve(null),
    db
      .select({ pinnedUrl: tenantSettings.pinnedUrl })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, device.organizationId)),
  ]);
  const effective = resolveEffectivePin({
    device: { pinMode: device.pinMode, pinnedUrl: device.pinnedUrl },
    store: storeRow,
    tenant: { pinnedUrl: ts?.pinnedUrl ?? null },
  });

  const ifNoneMatch = req.headers.get("if-none-match");
  const { version, notModified, payload } = await getDeviceConfig(
    device.organizationId,
    ifNoneMatch,
    { url: effective.url },
  );
```

Add imports: `store as storeTable, tenantSettings` to the schema import, and `import { resolveEffectivePin } from "@/lib/pin-resolve";`. The ETag/version already hashes the payload, so a store/tenant pin change naturally busts the device's 304.

- [ ] **Step 2: Gates + commit**

Run: `npx tsc --noEmit && npm test` — expected clean.

```bash
git add app/api/device/config/route.ts
git commit -m "feat(pin): device config serves the resolved effective pin"
```

---

### Task 7: Free inheritance on membership changes

**Files:**
- Modify: `lib/actions/devices.ts` (`reassignDevice` ~157, `unassignDevice` ~325, `assignDeviceToStore` ~358), `lib/actions/stores.ts` (`deleteStore` ~206, `deleteStoreForOrg` ~227), `lib/device-claim.ts` (`claimDevice` ~28)

**Interfaces:**
- Consumes: `pushEffectivePin(organizationId, deviceIds)` (Task 5).

- [ ] **Step 1: Device moves**

In each of `reassignDevice`, `unassignDevice`, `assignDeviceToStore`: after the `db.update(device)...` that changes `storeId` succeeds (and before/after the existing `recordAudit` — order vs audit doesn't matter), add:

```ts
  // Membership changed → re-deliver the (possibly different) effective pin. Free.
  await pushEffectivePin(organizationId, [deviceId]);
```

Import `pushEffectivePin` from `@/lib/pin-service`. Use each function's local names for org/device ids (read the surrounding code — e.g. some load the device row first; reuse its `organizationId`).

- [ ] **Step 2: Store deletion**

In `deleteStore` / `deleteStoreForOrg` (`lib/actions/stores.ts`): these move the store's devices to the pool. Capture the moved device ids (the update already targets `where storeId = X` — add `.returning({ id: device.id })` if not already collected), then after the store row is deleted:

```ts
  await pushEffectivePin(organizationId, movedDeviceIds);
```

- [ ] **Step 3: Claim**

In `claimDevice` (`lib/device-claim.ts`): after the claim update binds the device to a store, add the same `pushEffectivePin(organizationId, [deviceId])` call. (The device also fetches config on boot, which now resolves the effective pin — this covers an already-booted device being claimed.)

- [ ] **Step 4: Gates + commit**

Run: `npx tsc --noEmit && npm test` — expected clean.

```bash
git add lib/actions/devices.ts lib/actions/stores.ts lib/device-claim.ts
git commit -m "feat(pin): re-deliver effective pin on claim/move/unassign/store-delete (free)"
```

---

### Task 8: Data layer — pin overview + device pin context

**Files:**
- Modify: `lib/data.ts`

**Interfaces:**
- Consumes: `resolveEffectivePin`, `DevicePinRow`, `StorePinRow` (Tasks 3-4); Drizzle tables.
- Produces (consumed by the page in Task 10 and device detail in Task 11):

```ts
export interface PinOverview {
  tenant: { pinnedUrl: string | null; pinnedAt: string | null; reach: number }; // reach = devices that resolve at tenant level
  stores: {
    id: string; name: string; pinMode: PinMode; pinnedUrl: string | null;
    deviceCount: number; inheritingCount: number; effectiveUrl: string | null;
  }[];
  poolInheritingCount: number;
  exceptions: {
    id: string; name: string; storeId: string | null; storeName: string | null;
    pinMode: "custom" | "none"; pinnedUrl: string | null;
  }[];
}
export async function getPinOverview(organizationId: string): Promise<PinOverview>
export interface DevicePinContext {
  pinMode: PinMode;
  inheritedUrl: string | null;                 // what "inherit" would show
  inheritedSource: "store" | "tenant" | null;
}
export async function getDevicePinContext(organizationId: string, deviceId: string): Promise<DevicePinContext | null>
```

- [ ] **Step 1: Implement `getPinOverview`**

Add near the other tenant fns in `lib/data.ts`. Load all org devices (`id, name, storeId, pinMode, pinnedUrl`), all stores (`id, name, pinMode, pinnedUrl`), and `tenantSettings.pinnedUrl/pinnedAt`; then compute in JS with `resolveEffectivePin`:

```ts
export async function getPinOverview(organizationId: string): Promise<PinOverview> {
  const [devices, stores, [ts]] = await Promise.all([
    db
      .select({ id: device.id, name: device.name, storeId: device.storeId, pinMode: device.pinMode, pinnedUrl: device.pinnedUrl })
      .from(device)
      .where(eq(device.organizationId, organizationId)),
    db
      .select({ id: store.id, name: store.name, pinMode: store.pinMode, pinnedUrl: store.pinnedUrl })
      .from(store)
      .where(eq(store.organizationId, organizationId)),
    db
      .select({ pinnedUrl: tenantSettings.pinnedUrl, pinnedAt: tenantSettings.pinnedAt })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, organizationId)),
  ]);
  const tenantPinnedUrl = ts?.pinnedUrl ?? null;
  const storeById = new Map(stores.map((s) => [s.id, s]));
  const resolve = (d: (typeof devices)[number]) =>
    resolveEffectivePin({
      device: d,
      store: d.storeId ? (storeById.get(d.storeId) ?? null) : null,
      tenant: { pinnedUrl: tenantPinnedUrl },
    });

  // reach = devices whose chain delegates to the tenant level (counted even
  // when no tenant pin is set — it's the cost preview for setting one).
  const reachesTenant = (d: (typeof devices)[number]) => {
    if (d.pinMode !== "inherit") return false;
    const s = d.storeId ? storeById.get(d.storeId) : null;
    return !s || s.pinMode === "inherit";
  };

  return {
    tenant: {
      pinnedUrl: tenantPinnedUrl,
      pinnedAt: ts?.pinnedAt ? ts.pinnedAt.toISOString() : null,
      reach: devices.filter(reachesTenant).length,
    },
    stores: stores.map((s) => {
      const members = devices.filter((d) => d.storeId === s.id);
      const inheriting = members.filter((d) => d.pinMode === "inherit");
      return {
        id: s.id,
        name: s.name,
        pinMode: s.pinMode,
        pinnedUrl: s.pinnedUrl,
        deviceCount: members.length,
        inheritingCount: inheriting.length,
        effectiveUrl:
          inheriting.length > 0 ? resolve(inheriting[0]).url
          : s.pinMode === "custom" ? s.pinnedUrl
          : s.pinMode === "none" ? null
          : tenantPinnedUrl,
      };
    }),
    poolInheritingCount: devices.filter((d) => d.storeId === null && d.pinMode === "inherit").length,
    exceptions: devices
      .filter((d) => d.pinMode !== "inherit")
      .map((d) => ({
        id: d.id,
        name: d.name,
        storeId: d.storeId,
        storeName: d.storeId ? (storeById.get(d.storeId)?.name ?? null) : null,
        pinMode: d.pinMode as "custom" | "none",
        pinnedUrl: d.pinnedUrl,
      })),
  };
}
```

Imports to add at the top of `lib/data.ts` if missing: `resolveEffectivePin` from `@/lib/pin-resolve`, `PinMode` type from `@/lib/pin`, and export the two interfaces above.

- [ ] **Step 2: Implement `getDevicePinContext`**

```ts
export async function getDevicePinContext(
  organizationId: string,
  deviceId: string,
): Promise<DevicePinContext | null> {
  const [d] = await db
    .select({ id: device.id, storeId: device.storeId, pinMode: device.pinMode, pinnedUrl: device.pinnedUrl })
    .from(device)
    .where(and(eq(device.id, deviceId), eq(device.organizationId, organizationId)))
    .limit(1);
  if (!d) return null;
  const [storeRow, [ts]] = await Promise.all([
    d.storeId
      ? db
          .select({ pinMode: store.pinMode, pinnedUrl: store.pinnedUrl })
          .from(store)
          .where(eq(store.id, d.storeId))
          .limit(1)
          .then((r) => r[0] ?? null)
      : Promise.resolve(null),
    db
      .select({ pinnedUrl: tenantSettings.pinnedUrl })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, organizationId)),
  ]);
  // What "inherit" WOULD show: resolve with the device forced to inherit.
  const inherited = resolveEffectivePin({
    device: { pinMode: "inherit", pinnedUrl: null },
    store: storeRow,
    tenant: { pinnedUrl: ts?.pinnedUrl ?? null },
  });
  return {
    pinMode: d.pinMode,
    inheritedUrl: inherited.url,
    inheritedSource: inherited.source === "store" || inherited.source === "tenant" ? inherited.source : null,
  };
}
```

- [ ] **Step 3: Gates + commit**

Run: `npx tsc --noEmit && npm test` — expected clean.

```bash
git add lib/data.ts
git commit -m "feat(pin): data layer — getPinOverview + getDevicePinContext"
```

---

### Task 9: Server actions for all scopes

**Files:**
- Modify: `lib/actions/pin.ts`

**Interfaces:**
- Consumes: `applyScopedPinChange` (Task 5), `validatePinBody`/`PinMode` (`lib/pin.ts`), `requireTenant`, `canManageTenant`.
- Produces (consumed by UI Tasks 10-11):

```ts
export interface ScopedPinActionResult {
  ok: boolean; error?: string;
  affectedDevices?: number; creditsCharged?: number; pinnedUrl?: string | null;
}
export async function setOrgPinAction(url: string): Promise<ScopedPinActionResult>
export async function clearOrgPinAction(): Promise<ScopedPinActionResult>
export async function setStorePinAction(storeId: string, url: string): Promise<ScopedPinActionResult>
export async function setStorePinModeAction(storeId: string, mode: "none" | "inherit"): Promise<ScopedPinActionResult>
export async function setDevicePinAction(deviceId: string, url: string): Promise<PinActionResult>       // existing name, now via core
export async function clearDevicePinAction(deviceId: string): Promise<PinActionResult>                   // existing name, → inherit
export async function setDevicePinModeAction(deviceId: string, mode: "none" | "inherit"): Promise<ScopedPinActionResult>
```

- [ ] **Step 1: Refactor to a shared guard + add the new actions**

Rewrite `lib/actions/pin.ts` keeping the existing exports' names/signatures. Shared pieces: a `guard()` helper doing `requireTenant` + `canManageTenant`; ownership checks (store belongs to org for store actions — `db.select from store where id+organizationId`; device via the existing `loadTenantDevice`); then `applyScopedPinChange` with `actor: { type: "user", id, label: email }, via: "ui"`. Revalidation:

```ts
function revalidatePinSurfaces() {
  revalidatePath("/tenant/pinned-qr");
  revalidatePath("/tenant/devices");
  revalidatePath("/tenant/stores");
}
```

plus the existing `revalidateDevicePages(storeId, deviceId)` for device-scope actions. Full new action bodies follow this template (repeat per action with the right `ScopedPinChange`):

```ts
export async function setOrgPinAction(url: string): Promise<ScopedPinActionResult> {
  const g = await guard();
  if (!g.ok) return { ok: false, error: g.error };
  const v = validatePinBody({ url });
  if (!v.ok) return { ok: false, error: v.error };
  const res = await applyScopedPinChange({
    organizationId: g.organizationId,
    change: { scope: "org", url: v.url },
    actor: { type: "user", id: g.ctx.user.id, label: g.ctx.user.email },
    via: "ui",
    createdByUserId: g.ctx.user.id,
  });
  if (!res.ok) {
    return { ok: false, error: `Not enough credits — this change needs ${res.required}. Top up from Billing.` };
  }
  revalidatePinSurfaces();
  return { ok: true, affectedDevices: res.affectedDevices, creditsCharged: res.creditsCharged, pinnedUrl: v.url };
}
```

- `clearOrgPinAction` → change `{ scope: "org", url: null }`, no validation.
- `setStorePinAction(storeId, url)` → verify store ownership, change `{ scope: "store", storeId, mode: "custom", url }`.
- `setStorePinModeAction(storeId, mode)` → change `{ scope: "store", storeId, mode, url: null }`.
- `setDevicePinAction` / `clearDevicePinAction` → keep signatures/return type (`PinActionResult`), body switches to `applyScopedPinChange` with device scope (`mode:"custom"` + url / `mode:"inherit"` + null); keep `revalidateDevicePages` and add `revalidatePath("/tenant/pinned-qr")`.
- `setDevicePinModeAction(deviceId, mode)` → device ownership check, change `{ scope: "device", deviceId, mode, url: null }`.

- [ ] **Step 2: Gates + commit**

Run: `npx tsc --noEmit && npm test` — expected clean.

```bash
git add lib/actions/pin.ts
git commit -m "feat(pin): scoped server actions — org/store pin + mode changes"
```

---

### Task 10: `/tenant/pinned-qr` page + nav

**Files:**
- Modify: `lib/nav.ts` (TENANT_NAV)
- Create: `app/(tenant)/tenant/pinned-qr/page.tsx`, `components/pin/org-pin-card.tsx`, `components/pin/store-pin-table.tsx`

**Interfaces:**
- Consumes: `getPinOverview`, `getOrgQrStyle` (`lib/data.ts:1103`), `getCreditBalance` (`lib/credits.ts:38`), actions from Task 9, `QrSvg` + `qrCornerRadiusPx`/`qrShadowBoxShadow` (`components/qr-svg.tsx`, `lib/qr-svg.ts`), `PageHeader`/`SectionHeader`/`PageSection`, `canManageTenant`.

- [ ] **Step 1: Nav entry**

`lib/nav.ts` — add `Pin` to the lucide import and insert after "Device Settings":

```ts
  { label: "Pinned QR", href: "/tenant/pinned-qr", icon: Pin },
```

- [ ] **Step 2: Page (server component)**

`app/(tenant)/tenant/pinned-qr/page.tsx` — follow the devices-page pattern (`app/(tenant)/tenant/devices/page.tsx`): fragment return, `requireTenant`, `canManageTenant`. Load `getPinOverview`, `getOrgQrStyle`, `getCreditBalance` in `Promise.all`. Render:

1. `PageHeader title="Pinned QR" description="One QR your screens show while idle — tenant-wide, per store, or per device."`
2. `<OrgPinCard …/>` — pass `tenant` overview slice, `reach`, `qrStyle`, `creditsAvailable`, `canManage`.
3. `PageSection` "Stores" with `<StorePinTable stores={overview.stores} tenantPinnedUrl={overview.tenant.pinnedUrl} creditsAvailable canManage />`. Under it, if `poolInheritingCount > 0`, a one-line `text-sm text-muted-foreground` note: `"{n} unassigned device(s) follow the tenant-wide pin."`
4. `PageSection` "Device exceptions" — server-rendered `Table` of `overview.exceptions` (Device, Store, Mode badge `Custom`/`None`, URL mono truncated) where the device name links to `/tenant/stores/${storeId}/${id}` (pool devices: `/tenant/devices`). Empty state: `"No devices override their store or tenant pin."`

- [ ] **Step 3: `components/pin/org-pin-card.tsx` (client)**

Model it directly on `components/device-pin-control.tsx` (same Card + Dialog + `useTransition` + toast structure and the same `QrSvg`/`qrCornerRadiusPx`/`qrShadowBoxShadow`/`PIN_QR_DIM_PX = 128` preview treatment). Differences:

- Calls `setOrgPinAction` / `clearOrgPinAction`.
- Dialog copy shows scope cost: `Pinning tenant-wide updates up to {reach} device(s) — up to {reach} credit(s) (you have {creditsAvailable}). Removing is free.` Submit disabled when `reach > creditsAvailable` and the draft differs from the current URL.
- On success toast includes the real charge from the action result: `` `Pinned on ${res.affectedDevices} device(s) — ${res.creditsCharged} credit(s)` ``.
- Empty state copy: `"No tenant-wide pin. Devices fall back to their store pin or idle screen."`

- [ ] **Step 4: `components/pin/store-pin-table.tsx` (client)**

A `Card` containing a `Table` (columns: Store, Mode, Pinned URL, Devices, empty header for actions). Mode rendered as a `Badge` (`variant="secondary"` for Inherit, default for Custom, `outline` for None). Per row (when `canManage`) a "Manage" `Button size="sm" variant="outline"` opening ONE shared `Dialog` (state: `activeStore`). Dialog contents:

- A three-way mode choice rendered as three `Button`s (or shadcn `Tabs`): **Inherit** (`Follows the tenant-wide pin`), **Custom** (URL `Input` appears), **None** (`This store's devices show no pin`).
- Save handler: mode `custom` → `setStorePinAction(storeId, draftUrl)`; otherwise `setStorePinModeAction(storeId, mode)`.
- Cost line only for custom saves: `Up to {inheritingCount} device(s) — up to {inheritingCount} credit(s) (you have {creditsAvailable}).` Mode changes note `"Free."`
- `useTransition` + sonner toasts, mirroring `DevicePinControl`.

- [ ] **Step 5: Manual check + gates**

Run: `npm run dev`, sign in as `dana@roastwell.co` / `123456`, open `/tenant/pinned-qr`: set a tenant pin (watch credits drop by reach), set a store to Custom/None, verify the exceptions table lists custom/none devices. Then `npx tsc --noEmit && npm test`.

- [ ] **Step 6: Commit**

```bash
git add lib/nav.ts "app/(tenant)/tenant/pinned-qr" components/pin
git commit -m "feat(pin): /tenant/pinned-qr page — org pin card, store table, exceptions"
```

---

### Task 11: Tri-state `DevicePinControl` + device detail wiring

**Files:**
- Modify: `components/device-pin-control.tsx`, `app/(tenant)/tenant/stores/[storeId]/[deviceId]/page.tsx` (~lines 9-12, 60, 203-215)

**Interfaces:**
- Consumes: `getDevicePinContext` (Task 8), `setDevicePinModeAction` + existing pin actions (Task 9).
- Produces: `DevicePinControl` gains required props `pinMode: PinMode`, `inheritedUrl: string | null`, `inheritedSource: "store" | "tenant" | null`.

- [ ] **Step 1: Extend the component**

`components/device-pin-control.tsx`:

- Add the three new props; add local state `const [mode, setMode] = useState(props.pinMode)`.
- Render logic: `mode === "custom"` → today's pinned view (QR preview + URL + "Pinned … ago"). `mode === "inherit"` → if `inheritedUrl`, the same QR preview but labeled `Inherited from the {inheritedSource} pin` (no per-device Remove for inherited pins); else the existing "No pinned QR…" empty copy. `mode === "none"` → `"Pin disabled for this device — it always shows its idle screen."`
- Controls (when `canManage`): keep the Set/Change dialog (unchanged, calls `setDevicePinAction` → mode becomes `custom` on success). Replace the single Remove button with:
  - mode `custom`: `Remove` → `clearDevicePinAction(deviceId)` → on success `setMode("inherit")`, clear url state, toast `"Pin removed — device now follows the store/tenant pin"`.
  - mode `inherit` && `inheritedUrl`: `Don't pin here` (PinOff icon) → `setDevicePinModeAction(deviceId, "none")` → `setMode("none")`.
  - mode `none`: `Re-enable inherit` → `setDevicePinModeAction(deviceId, "inherit")` → `setMode("inherit")`.

- [ ] **Step 2: Wire the device detail page**

In `app/(tenant)/tenant/stores/[storeId]/[deviceId]/page.tsx`: add `getDevicePinContext` to the `@/lib/data` import, load it alongside the existing fetches (`const pinCtx = await getDevicePinContext(organizationId, deviceId)` — join the existing `Promise.all` if there is one), and pass `pinMode={pinCtx?.pinMode ?? "inherit"} inheritedUrl={pinCtx?.inheritedUrl ?? null} inheritedSource={pinCtx?.inheritedSource ?? null}` to `<DevicePinControl …>` (line ~203).

- [ ] **Step 3: Manual check + gates**

Dev server: on a device page, cycle Custom → Remove (shows inherited store/tenant pin if set) → Don't pin here → Re-enable. Then `npx tsc --noEmit && npm test`.

- [ ] **Step 4: Commit**

```bash
git add components/device-pin-control.tsx "app/(tenant)/tenant/stores/[storeId]/[deviceId]/page.tsx"
git commit -m "feat(pin): tri-state device pin card (inherit / custom / none)"
```

---

### Task 12: Public API — org + store endpoints, device endpoint rework

**Files:**
- Create: `lib/api/pin-idempotency.ts`, `app/api/v1/org/pin/route.ts`, `app/api/v1/stores/[storeId]/pin/route.ts`
- Modify: `app/api/v1/devices/[deviceId]/pin/route.ts`

**Interfaces:**
- Consumes: `applyScopedPinChange` (Task 5), `validatePinPutBody` (Task 2), `guardApiRequest`, `apiError`/`apiJson`, `hasScope(…, "devices:pin")`, `isOrgArchived`.
- Produces: response bodies —
  - org: `{ pin: {url, pinnedAt} | null, affectedDevices, creditsCharged }`
  - store: `{ storeId, pinMode, pin: {url, pinnedAt} | null, affectedDevices, creditsCharged }`
  - device: `{ deviceId, pinMode, pin: {url, pinnedAt} | null, effectiveUrl, affectedDevices, creditsCharged }`

- [ ] **Step 1: Extract the idempotency dance** (`lib/api/pin-idempotency.ts`)

Lift the claim/replay/release pattern from `app/api/v1/devices/[deviceId]/pin/route.ts:74-126` verbatim into three helpers (same table, same semantics — namespace passed in by the caller):

```ts
// lib/api/pin-idempotency.ts
// Shared Idempotency-Key claim/replay/release for the paid pin PUTs. The
// apiIdempotency table is shared with /trigger, so every endpoint namespaces
// its keys ("pin:" | "storepin:" | "orgpin:") — a key reused across endpoints
// must never replay another endpoint's stored body.
import { and, eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { apiIdempotency } from "@/lib/db/schema";

export type IdemClaim =
  | { owned: true; nsKey: string }
  | { owned: false; replay: { status: number; body: unknown } | null }
  | { owned: true; nsKey: null }; // no key requested

export async function claimPinIdempotency(a: {
  req: Request;
  namespace: "pin" | "storepin" | "orgpin";
  organizationId: string;
  placeholderBody: unknown;
}): Promise<IdemClaim> {
  const idemKey = a.req.headers.get("idempotency-key")?.trim() || null;
  if (!idemKey) return { owned: true, nsKey: null };
  const nsKey = `${a.namespace}:${idemKey}`;
  const claim = await db
    .insert(apiIdempotency)
    .values({ key: nsKey, organizationId: a.organizationId, responseStatus: 200, responseBody: a.placeholderBody, commandId: null })
    .onConflictDoNothing()
    .returning({ key: apiIdempotency.key });
  if (claim.length > 0) return { owned: true, nsKey };
  const [existing] = await db
    .select()
    .from(apiIdempotency)
    .where(and(eq(apiIdempotency.key, nsKey), eq(apiIdempotency.organizationId, a.organizationId)))
    .limit(1);
  return { owned: false, replay: existing ? { status: existing.responseStatus, body: existing.responseBody } : null };
}

export async function releasePinIdempotency(nsKey: string, organizationId: string): Promise<void> {
  await db.delete(apiIdempotency).where(and(eq(apiIdempotency.key, nsKey), eq(apiIdempotency.organizationId, organizationId)));
}

export async function storePinIdempotentResponse(nsKey: string, organizationId: string, body: unknown): Promise<void> {
  await db.update(apiIdempotency).set({ responseBody: body }).where(and(eq(apiIdempotency.key, nsKey), eq(apiIdempotency.organizationId, organizationId)));
}
```

- [ ] **Step 2: Org route** (`app/api/v1/org/pin/route.ts`)

Shape mirrors the device route: `guardApiRequest` → `requirePinScope` (copy the small helper — it selects the key's scopes and checks `devices:pin`) → parse JSON → `validatePinPutBody(raw, { allowMode: false })` → `isOrgArchived` gate (PUT only) → idempotency claim (`namespace: "orgpin"`, placeholder body `{ pin: { url, pinnedAt: new Date().toISOString() }, affectedDevices: 0, creditsCharged: 0 }`) → `applyScopedPinChange({ organizationId: auth.organizationId, change: { scope: "org", url: v.url }, actor: { type: "system" }, via: "api" })`.

- `!res.ok` → release claim, `apiError("insufficient_credits", \`Not enough credits — this change needs ${res.required}.\`, 402)`.
- Success → body `{ pin: { url: v.url, pinnedAt: (res.pinnedAt ?? new Date()).toISOString() }, affectedDevices: res.affectedDevices, creditsCharged: res.creditsCharged }`, store it on the claim, `apiJson(body, 200)`.
- `DELETE` → scope check, `applyScopedPinChange` with `{ scope: "org", url: null }` (no idempotency, no archive gate — clears are free/allowed), respond `{ pin: null, affectedDevices: res.ok ? res.affectedDevices : 0, creditsCharged: 0 }`.
- `export const runtime = "nodejs";`

- [ ] **Step 3: Store route** (`app/api/v1/stores/[storeId]/pin/route.ts`)

Same skeleton; `params: Promise<{ storeId: string }>`. Extra: ownership check —

```ts
const [s] = await db.select().from(storeTable).where(eq(storeTable.id, storeId)).limit(1);
if (!s || s.organizationId !== auth.organizationId) return apiError("store_not_found", "Store not found.", 404);
```

`validatePinPutBody(raw)` (modes allowed). `kind === "url"` → paid path with idempotency (`namespace: "storepin"`) + archive gate, change `{ scope: "store", storeId, mode: "custom", url: v.url }`. `kind === "mode"` → free: skip idempotency/archive-gate, change `{ scope: "store", storeId, mode: v.mode, url: null }`. `DELETE` → change `{ scope: "store", storeId, mode: "inherit", url: null }`. All responses `{ storeId, pinMode: <resulting mode>, pin: mode==="custom" ? {url, pinnedAt} : null, affectedDevices, creditsCharged }`.

- [ ] **Step 4: Rework the device route** (`app/api/v1/devices/[deviceId]/pin/route.ts`)

- Replace the inline idempotency block (lines 74-126) with the Task-12 helpers (`namespace: "pin"` — existing stored keys keep replaying correctly).
- `PUT`: switch body parsing to `validatePinPutBody(raw)`. `kind === "url"` → keep today's flow but call `applyScopedPinChange` (device scope, `mode: "custom"`) instead of `setDevicePin`; the free same-URL no-op short-circuit (lines 68-72) stays. `kind === "mode"` → free path, no idempotency/archive gate, `applyScopedPinChange` with the mode.
- `DELETE`: call `applyScopedPinChange` with `{ scope: "device", deviceId, mode: "inherit", url: null }` — **intentional semantics change** (spec §5): falls back to store/tenant pin; update the route header comment to say so.
- Responses now `{ deviceId, pinMode, pin, effectiveUrl, affectedDevices, creditsCharged }` — compute `effectiveUrl` by re-selecting the device row + `resolveEffectivePin` (or from `applyScopedPinChange`'s plan: for device scope, `affected[0]?.newUrl ?? <unchanged current effective>`; simplest correct: reuse `getDevicePinContext`-style resolution inline).
- Delete the now-unused `setDevicePin`/`clearDevicePin` wrappers from `lib/pin-service.ts` **if** `lib/actions/pin.ts` no longer imports them either (Task 9 moved actions to the core); `grep -rn "setDevicePin\|clearDevicePin" app lib` must come back empty before deleting.

- [ ] **Step 5: Manual smoke + gates**

With a `devices:pin` API key (create from `/tenant/api`):

```bash
curl -X PUT localhost:3000/api/v1/org/pin -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{"url":"https://example.com/campaign"}'
# expect 200 {pin:{url…}, affectedDevices:N, creditsCharged:N}
curl -X PUT localhost:3000/api/v1/stores/$STORE/pin -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{"mode":"none"}'
# expect 200 {storeId, pinMode:"none", pin:null, affectedDevices:…, creditsCharged:0}
curl -X DELETE localhost:3000/api/v1/devices/$DEV/pin -H "Authorization: Bearer $KEY"
# expect 200 {deviceId, pinMode:"inherit", …}
```

Then `npx tsc --noEmit && npm test`.

- [ ] **Step 6: Commit**

```bash
git add lib/api/pin-idempotency.ts app/api/v1/org "app/api/v1/stores/[storeId]/pin" "app/api/v1/devices/[deviceId]/pin/route.ts" lib/pin-service.ts
git commit -m "feat(pin): org + store pin API endpoints; device DELETE now falls back to inherit"
```

---

### Task 13: Final gates + verification sweep

**Files:** none new.

- [ ] **Step 1: Full gates**

Run: `npm test` (expect all suites green, including the new pin-resolve/pin tests), `npx tsc --noEmit`, `npm run build` (expect clean production build).

- [ ] **Step 2: End-to-end manual pass (dev server)**

1. `/tenant/pinned-qr`: set tenant pin → credits drop by reach; store → Custom URL; store → None; clear tenant pin (free).
2. Device page: verify the card shows "Inherited from the store pin"; Don't pin here → exceptions table on `/tenant/pinned-qr` lists it.
3. Move a device between stores (`/tenant/devices` or store page) → verify a fresh `pin` command row appears for it (`deviceCommand` table via `npm run db:studio` or the device page's command list).
4. Member account (invite or seed): `/tenant/pinned-qr` shows read-only state (no Manage buttons).

- [ ] **Step 3: Leave for user testing**

Do NOT deploy. Report gate results; the user tests locally (and on-device b580) and batch-deploys.

---

## Self-review notes (already applied)

- Spec §3 money-rule wording was tightened before planning (paid ⇔ URL set); the planner's `chargedCount` encodes exactly that.
- Audit constants here (`devicePinModeNone`, `storePinModeNone`) refine the spec's `devicePinModeChanged` — `none` is the only mode change worth flagging distinctly; `inherit` restorations log as `*PinCleared`.
- `setDevicePin`/`clearDevicePin` wrappers exist only between Tasks 5 and 12; Task 12 Step 4 deletes them once nothing imports them.
- Route/API layers are not unit-tested — consistent with this repo (vitest covers pure lib only); correctness there rides on the pure planner/resolver tests + the manual smoke steps.
