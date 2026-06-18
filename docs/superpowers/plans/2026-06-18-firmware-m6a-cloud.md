# M6a Cloud — Device claim endpoint + create-or-bind Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Cloud side of device-first provisioning — a device-generated pairing code can be claimed in the dashboard (create-or-bind) and the device fetches its key once via a new unauthenticated `GET /api/device/claim`.

**Architecture:** Add a one-time `device.pendingDeviceKey` column. Refactor `claimDevice` to create-or-bind and stash the raw key in `pendingDeviceKey` while keeping `pairingCode`. New `GET /api/device/claim?code=` returns `pending`/`claimed`+key (delivering once, then consuming the code). The poll decision is a pure, unit-tested helper; DB glue is verified by curl.

**Tech Stack:** Next.js (App Router, route handlers), Drizzle ORM over Neon, vitest, existing `lib/rate-limit.ts`, `lib/ids.ts`.

This is **Plan 1 of 2** for M6a. Plan 2 (firmware) consumes this endpoint and is written separately after this lands. Spec: `docs/superpowers/specs/2026-06-18-firmware-m6a-provisioning-design.md`.

---

### Task 1: Add `pendingDeviceKey` column to the device table

**Files:**
- Modify: `lib/db/schema.ts` (the `device` table, near `deviceKeyHash`)
- Generate: `lib/db/migrations/*` (via drizzle-kit)

- [ ] **Step 1: Add the column to the schema**

In `lib/db/schema.ts`, in the `device` table definition, add this line immediately after the `deviceKeyHash` column:

```ts
  // Raw device key held ONLY between claim and the device's first claim-poll fetch;
  // nulled on delivery (we otherwise store only deviceKeyHash). M6a provisioning.
  pendingDeviceKey: text("pending_device_key"),
```

- [ ] **Step 2: Generate the migration**

Run: `npm run db:generate`
Expected: a new SQL file under `lib/db/migrations/` adding `pending_device_key` to the `device` (table name `device`) — confirm it's `ADD COLUMN "pending_device_key" text;` and touches no other table.

- [ ] **Step 3: Apply the migration**

Run: `npm run db:migrate`
Expected: applies cleanly against Neon (`DATABASE_URL` from `.env.local`).

- [ ] **Step 4: Commit**

```bash
git add lib/db/schema.ts lib/db/migrations
git commit -m "feat(db): add device.pendingDeviceKey for one-time claim key delivery"
```

---

### Task 2: Pure claim-poll classifier + tests

**Files:**
- Create: `lib/provisioning.ts`
- Test: `lib/provisioning.test.ts`

- [ ] **Step 1: Write the failing test**

Create `lib/provisioning.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { classifyClaimPoll } from "./provisioning";

describe("classifyClaimPoll", () => {
  it("returns pending when no device row matches the code", () => {
    expect(classifyClaimPoll(null)).toEqual({ status: "pending", consume: false });
  });

  it("delivers the key and consumes when a claim is pending fetch", () => {
    expect(classifyClaimPoll({ pendingDeviceKey: "dvk_abc" })).toEqual({
      status: "claimed",
      deviceKey: "dvk_abc",
      consume: true,
    });
  });

  it("returns claimed without a key once already delivered", () => {
    expect(classifyClaimPoll({ pendingDeviceKey: null })).toEqual({
      status: "claimed",
      consume: false,
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run lib/provisioning.test.ts`
Expected: FAIL — cannot find module `./provisioning`.

- [ ] **Step 3: Write the implementation**

Create `lib/provisioning.ts`:

```ts
// Pure decision logic for the device claim-poll (GET /api/device/claim). Kept
// DB-free so it is unit-testable; the route handler does the DB lookup + applies
// the `consume` mutation.

export interface ClaimPollDecision {
  status: "pending" | "claimed";
  /** Present only on the one fetch that delivers the key. */
  deviceKey?: string;
  /** When true, the caller must null pendingDeviceKey + pairingCode (deliver once). */
  consume: boolean;
}

/**
 * Decide the claim-poll response from the matched device row (or null).
 * - no row            → pending (not claimed yet)
 * - pendingDeviceKey  → claimed + deliver the key, then consume
 * - key already gone  → claimed, no key
 */
export function classifyClaimPoll(
  device: { pendingDeviceKey: string | null } | null,
): ClaimPollDecision {
  if (!device) return { status: "pending", consume: false };
  if (device.pendingDeviceKey) {
    return { status: "claimed", deviceKey: device.pendingDeviceKey, consume: true };
  }
  return { status: "claimed", consume: false };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run lib/provisioning.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add lib/provisioning.ts lib/provisioning.test.ts
git commit -m "feat: classifyClaimPoll pure helper for device claim-poll"
```

---

### Task 3: Refactor `claimDevice` to create-or-bind

**Files:**
- Modify: `lib/receipts.ts` (the `claimDevice` function)

Current behavior: looks up the device by `pairingCode`, throws if missing, validates store org, mints a key, updates the row setting `pairingCode: null`, returns the raw key. New behavior: create-or-bind, stash the key in `pendingDeviceKey`, and **keep** `pairingCode` (the device still needs it to poll).

- [ ] **Step 1: Replace the `claimDevice` body**

In `lib/receipts.ts`, replace the entire `claimDevice` function with:

```ts
export async function claimDevice(pairingCode: string, storeId: string): Promise<ClaimResult> {
  const [store] = await db
    .select({ id: storeTable.id, organizationId: storeTable.organizationId })
    .from(storeTable)
    .where(eq(storeTable.id, storeId))
    .limit(1);
  if (!store) throw new Error("Store not found");

  const [existing] = await db
    .select()
    .from(deviceTable)
    .where(eq(deviceTable.pairingCode, pairingCode))
    .limit(1);
  if (existing?.claimedAt) throw new Error("Device already claimed");

  // Mint the key now; the raw key goes to pendingDeviceKey for the device's
  // one-time claim-poll fetch, only the hash is the durable credential.
  const { key, hash } = generateDeviceKey();

  if (existing) {
    // Bind a pre-seeded row (admin "Add device" path).
    if (store.organizationId !== existing.organizationId) {
      throw new Error("Store belongs to a different organization");
    }
    await db
      .update(deviceTable)
      .set({
        storeId,
        deviceKeyHash: hash,
        pendingDeviceKey: key,   // device fetches once via /api/device/claim
        claimedAt: new Date(),
        status: "offline",
        // pairingCode intentionally KEPT so the device can still poll by code.
      })
      .where(eq(deviceTable.id, existing.id));
    return { deviceId: existing.id, deviceName: existing.name, deviceKey: key };
  }

  // Create a row for a device-generated code with no pre-existing device.
  const deviceId = id("dev");
  const name = "New Printer";
  try {
    await db.insert(deviceTable).values({
      id: deviceId,
      organizationId: store.organizationId,
      storeId,
      name,
      status: "offline",
      connectionType: "wifi",
      firmwareVersion: "2.4.1",
      pairingCode,            // KEEP so the device can poll for its key
      deviceKeyHash: hash,
      pendingDeviceKey: key,
      claimedAt: new Date(),
      createdAt: new Date(),
    });
  } catch {
    // unique(pairingCode) violation → two devices generated the same code.
    throw new Error("Pairing code already in use");
  }
  return { deviceId, deviceName: name, deviceKey: key };
}
```

- [ ] **Step 2: Confirm imports exist**

Ensure `lib/receipts.ts` imports `id` from `./ids` (it already imports `generateDeviceKey` from `./ids`; add `id` to that import). Confirm `deviceTable` and `storeTable` are imported from `./db/schema`.

```ts
import { generateDeviceKey, id } from "./ids";
```

- [ ] **Step 3: Typecheck**

Run: `npx tsc --noEmit`
Expected: no errors (note: `pendingDeviceKey` exists from Task 1).

- [ ] **Step 4: Commit**

```bash
git add lib/receipts.ts
git commit -m "feat: claimDevice create-or-bind, stash one-time key, keep pairing code"
```

---

### Task 4: `GET /api/device/claim` endpoint

**Files:**
- Create: `app/api/device/claim/route.ts`

- [ ] **Step 1: Write the route**

Create `app/api/device/claim/route.ts`:

```ts
// GET /api/device/claim?code=<pairing-code> — UNAUTHENTICATED, code-gated, rate-limited.
// A provisioning device polls this until claimed, then receives its device key ONCE.
//   no row for code        → { status: "pending" }
//   pendingDeviceKey set    → { status: "claimed", deviceKey } then null key + code
//   key already delivered   → { status: "claimed" }

import { NextResponse } from "next/server";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { device as deviceTable } from "@/lib/db/schema";
import { checkRateLimit } from "@/lib/rate-limit";
import { classifyClaimPoll } from "@/lib/provisioning";

export const runtime = "nodejs";

export async function GET(req: Request) {
  const code = new URL(req.url).searchParams.get("code")?.trim();
  if (!code) return NextResponse.json({ error: "Missing code" }, { status: 400 });

  const rl = await checkRateLimit(`claim:${code}`, { limit: 30, windowMs: 60_000 });
  if (!rl.allowed) {
    return NextResponse.json(
      { error: "Too many requests" },
      { status: 429, headers: { "retry-after": String(Math.ceil(rl.retryAfterMs / 1000)) } },
    );
  }

  const [device] = await db
    .select({ id: deviceTable.id, pendingDeviceKey: deviceTable.pendingDeviceKey })
    .from(deviceTable)
    .where(eq(deviceTable.pairingCode, code))
    .limit(1);

  const decision = classifyClaimPoll(device ?? null);

  if (decision.consume && device) {
    await db
      .update(deviceTable)
      .set({ pendingDeviceKey: null, pairingCode: null })
      .where(eq(deviceTable.id, device.id));
  }

  return NextResponse.json(
    decision.deviceKey
      ? { status: decision.status, deviceKey: decision.deviceKey }
      : { status: decision.status },
  );
}
```

- [ ] **Step 2: Typecheck + build the route**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add app/api/device/claim/route.ts
git commit -m "feat(api): GET /api/device/claim — code-gated one-time key delivery"
```

---

### Task 5: Dashboard claim UX — de-emphasize the key

**Files:**
- Modify: `components/claim-device-dialog.tsx` (success state)

The device now fetches its own key, so the merchant shouldn't be told to copy/paste a key. Keep the key visible as a collapsible "advanced / manual setup" fallback, but lead with activation messaging.

- [ ] **Step 1: Change the success copy**

In `components/claim-device-dialog.tsx`, in the post-claim success branch (where `result.deviceKey` is shown), change the primary message to:

> **Device claimed.** It will activate automatically within a few seconds — watch the printer screen return to the home screen.

Move the raw `deviceKey` display behind a collapsed `<details>` labelled "Manual setup (advanced)" with the existing copy-key UI inside. Do not remove the key display — it's the hand-paste fallback.

- [ ] **Step 2: Verify the dialog renders**

Run: `npm run build`
Expected: build succeeds (no type/lint errors in the dialog).

- [ ] **Step 3: Commit**

```bash
git add components/claim-device-dialog.tsx
git commit -m "feat(ui): claim dialog leads with auto-activation, key as advanced fallback"
```

---

### Task 6: End-to-end verification (manual) + deploy

**Files:** none (verification).

- [ ] **Step 1: Run the app locally**

Run: `npm run dev`

- [ ] **Step 2: Poll before claim → pending**

Pick an unused human code, e.g. `TEST-CODE` (format doesn't matter for the endpoint).
Run: `curl -s "http://localhost:3000/api/device/claim?code=TEST-CODE"`
Expected: `{"status":"pending"}`

- [ ] **Step 3: Claim via the dashboard**

In the tenant UI (signed in as `dana@roastwell.co` / `123456`), open a store → "Claim printer" → enter `TEST-CODE`. The new device should be created and bound to that store (create path). The dialog shows the auto-activation message.

- [ ] **Step 4: Poll after claim → key delivered once**

Run: `curl -s "http://localhost:3000/api/device/claim?code=TEST-CODE"`
Expected: `{"status":"claimed","deviceKey":"dvk_..."}`

- [ ] **Step 5: Poll again → claimed, no key (consumed)**

Run: `curl -s "http://localhost:3000/api/device/claim?code=TEST-CODE"`
Expected: `{"status":"claimed"}` (no `deviceKey`; the code is now consumed — a third poll returns `{"status":"pending"}` because `pairingCode` was nulled, which is fine: the device already has its key).

- [ ] **Step 6: Confirm the device row in the DB**

Run: `npm run db:studio` (or a quick query) — the claimed device has `deviceKeyHash` set, `pendingDeviceKey` NULL, `pairingCode` NULL, `claimedAt` set, bound to the store.

- [ ] **Step 7: Run the full test suite**

Run: `npm run test`
Expected: all green (including `lib/provisioning.test.ts`).

- [ ] **Step 8: Deploy to production**

Per `[[vercel-deploy]]`: `vercel --prod` (the migration must be applied to the prod DB — run `npm run db:migrate` against the prod `DATABASE_URL` first, or confirm the deploy pipeline applies it). Verify `GET /api/device/claim?code=unused` returns `{"status":"pending"}` on `ditto-admin-brown.vercel.app`.

---

## Self-Review

**Spec coverage:**
- Data model (`pendingDeviceKey`) → Task 1. ✓
- `claimDevice` create-or-bind + keep code + pendingDeviceKey → Task 3. ✓
- `GET /api/device/claim` state machine (pending/claimed+key/claimed-no-key, consume) → Tasks 2 (logic) + 4 (route). ✓
- Rate-limited, unauthenticated → Task 4. ✓
- Dashboard UX change → Task 5. ✓
- Tests: pure classifier unit-tested (Task 2); create-or-bind + endpoint verified end-to-end via curl (Task 6), since this repo has no DB-test harness and mocking drizzle chains is brittle. The firmware HIL (Plan 2) is the final integration check. ✓ (deviation from "unit test claimDevice" noted intentionally)

**Placeholder scan:** none — every code step has full code; the only "advanced fallback" wording in Task 5 is intentional UX copy.

**Type consistency:** `pendingDeviceKey: string | null` used consistently across schema (Task 1), `classifyClaimPoll` (Task 2), `claimDevice` (Task 3), and the route select (Task 4). `ClaimResult` shape unchanged.

**Out of scope (Plan 2 / later):** firmware code-gen, claim-poll, NVS, pairingCode/steps widgets, boot state machine; on-screen Wi-Fi; pending-key TTL.
