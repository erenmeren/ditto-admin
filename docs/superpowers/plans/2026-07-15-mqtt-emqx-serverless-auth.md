# MQTT EMQX Serverless Auth (Path A) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the JWT-based MQTT auth (unsupported on EMQX Serverless) with EMQX built-in-database username/password auth, where each device authenticates to the broker using its existing device key, provisioned into EMQX at claim time.

**Architecture:** The `mqtt` config block loses its JWT password — the device uses its own device key (which it already fetches at claim) as the MQTT password (username = deviceId). At claim we provision `{user_id: deviceId, password: deviceKey}` into EMQX's built-in DB via the management API; on device delete/unclaim we deprovision. Per-device topic isolation is enforced by a single global built-in ACL rule using the `${clientid}` placeholder (console one-time). Everything stays env-gated: absent EMQX env → no-op, HTTP polling fallback.

**Tech Stack:** Next.js 16 (App Router, `runtime = "nodejs"`), Drizzle over neon-http, EMQX Cloud Serverless management REST API (`/api/v5`, Basic auth), vitest.

## Global Constraints

- **Additive / env-gated no-op, non-breaking.** No DB migration. With the EMQX env group absent, `mqttEnabled()` is false → config omits `mqtt`, provisioning fns are no-ops returning false, webhooks 503. Behavior identical to today.
- **Device key IS the MQTT password.** No new secret; the config block carries NO password (firmware uses its stored device key, same key as the HTTP Bearer credential). username = clientId = deviceId.
- **Provisioning is fail-open.** A provisioning/deprovisioning hiccup must NEVER fail a device claim/delete/offboard — wrap in try/catch exactly like the existing `syncDeviceSubscription` calls (`lib/documents.ts:72`, `lib/actions/devices.ts:223`). A device that failed to provision simply falls back to HTTP polling.
- **EMQX management API:** base = `EMQX_API_URL` (e.g. `https://xxxx.eu-central-1.emqxsl.com:8443/api/v5`), Basic auth `EMQX_API_KEY:EMQX_API_SECRET` (same as `buildPublishRequest` at `lib/mqtt.ts:75-76`). Built-in DB authenticator id path segment is the literal `password_based:built_in_database` (do NOT URL-encode the colon).
  - Create user: `POST {base}/authentication/password_based:built_in_database/users` body `{ user_id, password, is_superuser: false }`.
  - Update password: `PUT {base}/authentication/password_based:built_in_database/users/{user_id}` body `{ password }`.
  - Delete user: `DELETE {base}/authentication/password_based:built_in_database/users/{user_id}`.
- **MQTT_JWT_SECRET is removed** from the env schema, `.env.example`, `mqttEnabled()`, and all tests.
- **Tests:** pure/impure lib logic in `lib/mqtt.test.ts` (vitest, `include` = `lib/**/*.test.ts`); route/action wiring verified by `tsc` + full suite (no route/action unit tests in this repo). `npm test` = `vitest run`.
- **Commit** after each task with explicit `git add <files>` (never `-A` — an unrelated `lib/nav.ts` change must stay uncommitted). Branch: `feat/mqtt-transport` (already checked out).

---

## File Structure

**Modified:**
- `lib/mqtt.ts` — drop JWT (`SignJWT`/`jwtKey`/`mintDeviceMqttJwt`/`JWT_TTL_SECONDS`); `mqttEnabled()` drops the `MQTT_JWT_SECRET` check; `buildMqttConfigBlock` returns no password; add `provisionDeviceMqtt` + `deprovisionDeviceMqtt`.
- `lib/mqtt.test.ts` — remove JWT tests; update the `FULL` env fixture + the `buildMqttConfigBlock` and `mqttEnabled` tests; add `provisionDeviceMqtt`/`deprovisionDeviceMqtt` tests.
- `lib/env.ts` — remove `MQTT_JWT_SECRET`.
- `.env.example` — remove the `MQTT_JWT_SECRET` line.
- `package.json` — remove the now-unused direct `jose` dependency.
- `lib/documents.ts` — provision on claim (both branches of `claimDevice`).
- `lib/actions/devices.ts` — deprovision in `deleteDevice`.
- `lib/factory-registry.ts` — deprovision in the dealloc path that deletes the device (`:449`).
- `lib/actions/offboarding.ts` — deprovision each returned-to-stock device where keys are revoked.
- `docs/runbooks/emqx-setup.md` — rewrite auth section for built-in DB + global ACL placeholder; drop JWT; note device-key-as-password.

---

### Task 1: Swap JWT for device-key auth + provisioning helpers in `lib/mqtt.ts` — TDD

**Files:**
- Modify: `lib/mqtt.ts`, `lib/env.ts`, `.env.example`, `package.json`
- Test: `lib/mqtt.test.ts`

**Interfaces:**
- Consumes: `env` from `@/lib/env`.
- Produces (consumed by Tasks 2–3):
  - `mqttEnabled(): boolean` (no longer requires `MQTT_JWT_SECRET`)
  - `buildMqttConfigBlock(deviceId): Promise<{ host: string; port: number; clientId: string; username: string } | null>` (NO `password`)
  - `provisionDeviceMqtt(deviceId: string, mqttPassword: string): Promise<boolean>`
  - `deprovisionDeviceMqtt(deviceId: string): Promise<boolean>`
- Unchanged exports: `mqttEnabled`, `buildPublishRequest`, `publishCommand`, `verifyWebhookSecret`, the three parsers, `MqttCommand`.

- [ ] **Step 1: Update the tests first (TDD)**

In `lib/mqtt.test.ts`:
- Remove `MQTT_JWT_SECRET` from the `FULL` fixture object and remove the `import { jwtVerify } from "jose"` line.
- Delete the entire `describe("mintDeviceMqttJwt", ...)` block (including the disabled-throw test).
- In `describe("mqttEnabled", ...)`, change the "false when one required var is missing" case to omit a still-required var, e.g. `{ ...FULL, MQTT_BROKER_HOST: undefined }`.
- In `describe("buildMqttConfigBlock", ...)`, change the enabled assertion to expect NO password: `expect(block).toEqual({ host: "broker.example.com", port: 8883, clientId: "dev_1", username: "dev_1" })` and delete the JWT-shape assertion.
- Add two new describe blocks (stub `fetch` like the existing `publishCommand` tests, with `vi.stubGlobal("fetch", fetchSpy)` / `afterEach(() => vi.unstubAllGlobals())`):

```ts
describe("provisionDeviceMqtt", () => {
  it("is a no-op returning false when disabled", async () => {
    const fetchSpy = vi.fn();
    vi.stubGlobal("fetch", fetchSpy);
    expect(await provisionDeviceMqtt("dev_1", "pw")).toBe(false);
    expect(fetchSpy).not.toHaveBeenCalled();
    vi.unstubAllGlobals();
  });
  it("POSTs a built-in-db user and returns true on ok", async () => {
    Object.assign(ENV, FULL);
    const fetchSpy = vi.fn().mockResolvedValue({ ok: true, status: 200 } as Response);
    vi.stubGlobal("fetch", fetchSpy);
    expect(await provisionDeviceMqtt("dev_9", "secret")).toBe(true);
    expect(fetchSpy).toHaveBeenCalledTimes(1);
    const [url, opts] = fetchSpy.mock.calls[0];
    expect(url).toBe("https://broker.example.com:8443/api/v5/authentication/password_based:built_in_database/users");
    expect(opts.method).toBe("POST");
    expect(opts.headers.Authorization).toMatch(/^Basic /);
    expect(JSON.parse(opts.body)).toEqual({ user_id: "dev_9", password: "secret", is_superuser: false });
    vi.unstubAllGlobals();
  });
  it("on 409 conflict updates the password via PUT and returns true", async () => {
    Object.assign(ENV, FULL);
    const fetchSpy = vi.fn()
      .mockResolvedValueOnce({ ok: false, status: 409 } as Response)
      .mockResolvedValueOnce({ ok: true, status: 200 } as Response);
    vi.stubGlobal("fetch", fetchSpy);
    expect(await provisionDeviceMqtt("dev_9", "secret2")).toBe(true);
    expect(fetchSpy).toHaveBeenCalledTimes(2);
    const [url, opts] = fetchSpy.mock.calls[1];
    expect(url).toBe("https://broker.example.com:8443/api/v5/authentication/password_based:built_in_database/users/dev_9");
    expect(opts.method).toBe("PUT");
    expect(JSON.parse(opts.body)).toEqual({ password: "secret2" });
    vi.unstubAllGlobals();
  });
});

describe("deprovisionDeviceMqtt", () => {
  it("is a no-op returning false when disabled", async () => {
    const fetchSpy = vi.fn();
    vi.stubGlobal("fetch", fetchSpy);
    expect(await deprovisionDeviceMqtt("dev_1")).toBe(false);
    expect(fetchSpy).not.toHaveBeenCalled();
    vi.unstubAllGlobals();
  });
  it("DELETEs the user and treats 404 as success", async () => {
    Object.assign(ENV, FULL);
    const fetchSpy = vi.fn().mockResolvedValue({ ok: false, status: 404 } as Response);
    vi.stubGlobal("fetch", fetchSpy);
    expect(await deprovisionDeviceMqtt("dev_9")).toBe(true);
    const [url, opts] = fetchSpy.mock.calls[0];
    expect(url).toBe("https://broker.example.com:8443/api/v5/authentication/password_based:built_in_database/users/dev_9");
    expect(opts.method).toBe("DELETE");
    vi.unstubAllGlobals();
  });
});
```

Update the import line at the top of the file to include `provisionDeviceMqtt, deprovisionDeviceMqtt` and drop `mintDeviceMqttJwt`.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run lib/mqtt.test.ts`
Expected: FAIL — `provisionDeviceMqtt`/`deprovisionDeviceMqtt` are not exported yet (and the removed-JWT edits reference the new shape).

- [ ] **Step 3: Remove `MQTT_JWT_SECRET` from env**

In `lib/env.ts`, delete the `MQTT_JWT_SECRET` line (the `// HS256 signing secret...` comment + the `MQTT_JWT_SECRET: z.string().optional(),` line). In `.env.example`, delete the `MQTT_JWT_SECRET="..."` line.

- [ ] **Step 4: Rewrite the JWT parts of `lib/mqtt.ts`**

- Delete the `import { SignJWT } from "jose";` line, the `JWT_TTL_SECONDS` const, the `jwtKey()` fn, and the entire `mintDeviceMqttJwt` fn.
- Change `mqttEnabled()` to drop the `env.MQTT_JWT_SECRET &&` clause:

```ts
export function mqttEnabled(): boolean {
  return Boolean(
    env.EMQX_API_URL &&
      env.EMQX_API_KEY &&
      env.EMQX_API_SECRET &&
      env.EMQX_WEBHOOK_SECRET &&
      env.MQTT_BROKER_HOST,
  );
}
```

- Change `buildMqttConfigBlock` to drop the password:

```ts
/** The `mqtt` block for /api/device/config, or null when disabled. The device
 *  authenticates to the broker with its own device key as the MQTT password
 *  (username = deviceId), so no secret is carried here. */
export async function buildMqttConfigBlock(deviceId: string): Promise<{
  host: string;
  port: number;
  clientId: string;
  username: string;
} | null> {
  if (!mqttEnabled()) return null;
  return {
    host: env.MQTT_BROKER_HOST as string,
    port: Number(env.MQTT_BROKER_PORT ?? 8883),
    clientId: deviceId,
    username: deviceId,
  };
}
```

- Add a small shared helper for the EMQX management base + auth header (mirror `buildPublishRequest`), then the two provisioning fns:

```ts
function emqxAuthUsersBase(): string {
  return `${(env.EMQX_API_URL as string).replace(/\/$/, "")}/authentication/password_based:built_in_database/users`;
}

function emqxAuthHeader(): string {
  return `Basic ${Buffer.from(`${env.EMQX_API_KEY}:${env.EMQX_API_SECRET}`).toString("base64")}`;
}

/** Provision (or update) the device's built-in-DB MQTT credential. Idempotent:
 *  a 409 (user exists) updates the password via PUT. No-op false when disabled. */
export async function provisionDeviceMqtt(deviceId: string, mqttPassword: string): Promise<boolean> {
  if (!mqttEnabled()) return false;
  const usersUrl = emqxAuthUsersBase();
  const headers = { Authorization: emqxAuthHeader(), "Content-Type": "application/json" };
  try {
    const res = await fetch(usersUrl, {
      method: "POST",
      headers,
      body: JSON.stringify({ user_id: deviceId, password: mqttPassword, is_superuser: false }),
      signal: AbortSignal.timeout(2000),
    });
    if (res.ok) return true;
    if (res.status === 409) {
      const upd = await fetch(`${usersUrl}/${deviceId}`, {
        method: "PUT",
        headers,
        body: JSON.stringify({ password: mqttPassword }),
        signal: AbortSignal.timeout(2000),
      });
      return upd.ok;
    }
    return false;
  } catch {
    return false;
  }
}

/** Delete the device's built-in-DB MQTT credential. 404 (already gone) is
 *  treated as success. No-op false when disabled. */
export async function deprovisionDeviceMqtt(deviceId: string): Promise<boolean> {
  if (!mqttEnabled()) return false;
  try {
    const res = await fetch(`${emqxAuthUsersBase()}/${deviceId}`, {
      method: "DELETE",
      headers: { Authorization: emqxAuthHeader() },
      signal: AbortSignal.timeout(2000),
    });
    return res.ok || res.status === 404;
  } catch {
    return false;
  }
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `npx vitest run lib/mqtt.test.ts`
Expected: PASS (all remaining + new blocks green).

- [ ] **Step 6: Remove the now-unused direct `jose` dependency**

Run: `npm uninstall jose`
Expected: `jose` removed from `package.json` dependencies. (It remains available transitively via better-auth; nothing in our code imports it now — confirm with `grep -rn "from \"jose\"" lib app`, expected no hits.)

- [ ] **Step 7: Full suite + typecheck**

Run: `npm test`
Expected: PASS (currently 313; JWT tests removed and provisioning tests added — count will shift, all green).

Run: `npx tsc --noEmit`
Expected: no errors. (The config route spreads the block generically, so the dropped `password` field needs no route change — confirm the route still typechecks.)

- [ ] **Step 8: Commit**

```bash
git add lib/mqtt.ts lib/mqtt.test.ts lib/env.ts .env.example package.json package-lock.json
git commit -m "refactor(mqtt): device-key built-in-DB auth replacing JWT (EMQX Serverless)"
```

---

### Task 2: Provision the MQTT credential on claim

**Files:**
- Modify: `lib/documents.ts` (`claimDevice`, both branches)

**Interfaces:**
- Consumes: `provisionDeviceMqtt` from `@/lib/mqtt` (Task 1).

- [ ] **Step 1: Import the helper**

In `lib/documents.ts`, add to the imports: `import { provisionDeviceMqtt } from "@/lib/mqtt";`

- [ ] **Step 2: Provision after the bind-existing branch**

In `claimDevice`, the bind-existing branch returns at `lib/documents.ts:77`. Immediately before that `return`, after the `syncDeviceSubscription` try/catch, add (fail-open):

```ts
    // Provision the device's MQTT credential (device key = MQTT password).
    // Fail-open: a provisioning hiccup must never fail a claim — the device
    // just uses HTTP polling until it is reprovisioned.
    try {
      await provisionDeviceMqtt(existing.id, key);
    } catch (err) {
      console.error("mqtt provision after claim failed", err);
    }
```

- [ ] **Step 3: Provision after the create-new branch**

Similarly, before the final `return { deviceId, deviceName: name, deviceKey: key };` at `lib/documents.ts:113`, after that branch's `syncDeviceSubscription` try/catch, add the same block but with `deviceId` instead of `existing.id`:

```ts
    try {
      await provisionDeviceMqtt(deviceId, key);
    } catch (err) {
      console.error("mqtt provision after claim failed", err);
    }
```

- [ ] **Step 4: Typecheck + full suite**

Run: `npx tsc --noEmit` → no errors.
Run: `npm test` → all pass (no behavior change when MQTT disabled — `provisionDeviceMqtt` returns false without a fetch).

- [ ] **Step 5: Commit**

```bash
git add lib/documents.ts
git commit -m "feat(mqtt): provision device MQTT credential on claim"
```

---

### Task 3: Deprovision the MQTT credential on delete / dealloc / return-to-stock

**Files:**
- Modify: `lib/actions/devices.ts` (`deleteDevice`), `lib/factory-registry.ts` (dealloc-that-deletes, near `:449`), `lib/actions/offboarding.ts` (returned-to-stock key revocation)

**Interfaces:**
- Consumes: `deprovisionDeviceMqtt` from `@/lib/mqtt` (Task 1).

- [ ] **Step 1: deleteDevice**

In `lib/actions/devices.ts`, import `deprovisionDeviceMqtt` from `@/lib/mqtt`. In `deleteDevice`, after `await db.delete(deviceTable)...` (`:211`) and before/after the `syncDeviceSubscription` try/catch, add (fail-open):

```ts
  try {
    await deprovisionDeviceMqtt(deviceId);
  } catch (err) {
    console.error("mqtt deprovision after delete failed", err);
  }
```

- [ ] **Step 2: factory dealloc**

In `lib/factory-registry.ts`, import `deprovisionDeviceMqtt` from `@/lib/mqtt`. In the function that runs `await tx.delete(deviceTable).where(eq(deviceTable.id, deviceId));` (near `:449`), after the transaction completes (outside `tx`, in the returned result path, or immediately after the delete within the same function scope — do NOT put a network call inside the DB transaction), add a fail-open `deprovisionDeviceMqtt(deviceId)` call. If the delete happens inside `db.transaction(async (tx) => {...})`, capture `deviceId` and call `deprovisionDeviceMqtt` AFTER the transaction resolves.

```ts
  // after the transaction that deleted the device row resolves:
  try {
    await deprovisionDeviceMqtt(deviceId);
  } catch (err) {
    console.error("mqtt deprovision after dealloc failed", err);
  }
```

- [ ] **Step 3: offboarding return-to-stock**

In `lib/actions/offboarding.ts`, import `deprovisionDeviceMqtt` from `@/lib/mqtt`. Locate where returned-to-stock devices have their keys revoked (the `returnIds` from `partitionDispositions` — the mutation that nulls `deviceKeyHash`/`claimedAt` and feeds `summary.revokedKeys`). After that mutation succeeds, deprovision each returned device (fail-open, outside any DB transaction):

```ts
  for (const id of returnIds) {
    try {
      await deprovisionDeviceMqtt(id);
    } catch (err) {
      console.error("mqtt deprovision after offboard failed", err);
    }
  }
```

Read the surrounding code first to use the correct local variable name for the returned ids and to place the loop after the DB mutation, not inside a transaction callback.

- [ ] **Step 4: Typecheck + full suite**

Run: `npx tsc --noEmit` → no errors.
Run: `npm test` → all pass (disabled → deprovision is a no-op, no behavior change).

- [ ] **Step 5: Commit**

```bash
git add lib/actions/devices.ts lib/factory-registry.ts lib/actions/offboarding.ts
git commit -m "feat(mqtt): deprovision device MQTT credential on delete/dealloc/offboard"
```

---

### Task 4: Rewrite the EMQX runbook for built-in-DB auth

**Files:**
- Modify: `docs/runbooks/emqx-setup.md`

- [ ] **Step 1: Replace the JWT auth section (section 3) with built-in DB + global ACL**

Rewrite section 3 ("JWT authentication") to:

```markdown
## 3. Authentication — built-in database (Serverless does NOT support JWT)
- Console → Access Control → Authentication → add **Password-Based → Built-in Database**.
- Password hashing: default (bcrypt/sha256) is fine — the cloud provisions
  credentials via the management API; nothing to configure per-device here.
- Credentials are provisioned automatically by the app at device-claim time
  (`provisionDeviceMqtt` → `POST /api/v5/authentication/password_based:built_in_database/users`,
  `{user_id: deviceId, password: <device key>, is_superuser: false}`), and deleted
  on device delete/unclaim. **The device's MQTT password is its device key**
  (the same key it uses for HTTP `Authorization: Bearer`), username = deviceId.

## 3b. Authorization — one global ACL rule with a placeholder
- Console → Access Control → Authorization → **Built-in Database**.
- Add a single rule that confines every device to its own topics using the
  `${clientid}` placeholder:
  - allow **subscribe** to `d/${clientid}/cmd`
  - allow **publish** to `d/${clientid}/ack` and `d/${clientid}/hb`
  - (a catch-all deny is the default)
- Because username = clientId = deviceId, this one rule isolates every device
  with no per-device ACL provisioning.
- **Validation:** after setup, confirm device A cannot subscribe to
  `d/<deviceB>/cmd` (a leaked credential must not reach another device's topics).
  If the Serverless tier does not honor `${clientid}` placeholders in built-in
  authorization, fall back to per-device ACL provisioning via the management API
  in `provisionDeviceMqtt` (contingency — not built by default).
```

- [ ] **Step 2: Fix section 5 (env vars) and drop the JWT secret**

Remove any `MQTT_JWT_SECRET` mention. The required env group is now:
`EMQX_API_URL`, `EMQX_API_KEY`, `EMQX_API_SECRET`, `EMQX_WEBHOOK_SECRET`,
`MQTT_BROKER_HOST`, `MQTT_BROKER_PORT`. Keep the three Data-Integration webhooks
(section 4) unchanged, including the `{...payload, "clientid": clientid}` order note.

- [ ] **Step 3: Commit**

```bash
git add docs/runbooks/emqx-setup.md
git commit -m "docs(mqtt): runbook for EMQX Serverless built-in-DB auth + placeholder ACL"
```

---

### Task 5: Full verification & memory update

- [ ] **Step 1: Gates**

Run: `npm test` → all pass.
Run: `npx tsc --noEmit` → clean.
Run: `npm run build` → succeeds; confirm `/api/mqtt/{ack,heartbeat,presence}`, `/api/device/config`, `/api/v1/devices/[deviceId]/trigger` still registered.

- [ ] **Step 2: Confirm no-op default**

With no EMQX env: `mqttEnabled()` false → config omits `mqtt`; `provisionDeviceMqtt`/`deprovisionDeviceMqtt` return false without a fetch (covered by the Task 1 tests); claim/delete/offboard behave exactly as before.

- [ ] **Step 3: Update memory**

Update the `mqtt-transport` memory entry: JWT replaced by EMQX Serverless built-in-DB device-key auth (Path A); device key = MQTT password; provision on claim / deprovision on delete-dealloc-offboard; global `${clientid}` ACL placeholder; `MQTT_JWT_SECRET` removed. Remaining = create the EMQX Serverless deployment + App Key/Secret + built-in DB authenticator + the global ACL rule + three webhooks (runbook), re-claim b580 to provision it, then HIL.

---

## Self-Review Notes

- **Spec coverage:** JWT removal (Task 1), device-key-as-password config block (Task 1), provisioning API create/update/delete (Task 1), provision-on-claim (Task 2), deprovision on the three key-death paths (Task 3), env/`.env.example`/`jose` cleanup (Task 1), runbook + ACL placeholder + validation (Task 4), gates + memory (Task 5).
- **No DB migration; webhook routes, `publishCommand`, trigger publish, and the credit/ack path are all untouched** — Path A only changes how the device authenticates to the broker, not how commands or acks flow.
- **Fail-open everywhere** mirrors the existing `syncDeviceSubscription` pattern, so a broker/API outage never breaks a claim/delete/offboard.
- **Type consistency:** `provisionDeviceMqtt(deviceId, mqttPassword)` / `deprovisionDeviceMqtt(deviceId)` signatures are identical across Tasks 1–3; `buildMqttConfigBlock`'s return type drops `password` in Task 1 and no consumer references that field.
- **Migration caveat:** already-claimed devices (raw key not stored) can't be retro-provisioned — they must be re-claimed. Fleet is tiny; re-claim b580 during validation.
