# MQTT-only Device Transport Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move every recurring device↔cloud exchange onto MQTT and delete the HTTP device API outright, with no fallback path.

**Architecture:** Config and the OTA manifest travel on the existing `d/{id}/cmd` topic as payload-carrying `config-changed` / `firmware-update` commands. The single new topic is `d/{id}/cfg/get`, which the device publishes once per boot to ask for config; a new EMQX rule forwards it to `POST /api/mqtt/config-request`, and the cloud answers by publishing a freshly-presigned config. Work runs in four ordered phases across two repos: cloud-additive → firmware → OTA convergence → cloud-subtractive.

**Tech Stack:** Next.js 16 App Router, Drizzle ORM over Neon, vitest (pure-function tests in `lib/*.test.ts`), EMQX Cloud Serverless HTTP Publish + webhook rules, ESP-IDF 5.5.4 / esp-mqtt / cJSON / LVGL.

## Global Constraints

- **Two repos.** Cloud tasks run in `/home/meren/projects/ditto-admin`. Firmware tasks run in `/home/meren/projects/ditto-firmware` (ESP-IDF export first: `. $HOME/.espressif/v5.5/esp-idf/export.sh`).
- **Phase order is load-bearing.** Phase C must not start until Phase B2 confirms the device converged. Running C early cuts the device's config *and* OTA paths at once, leaving USB recovery as the only way back.
- **`.env.local` points at PROD Neon** (org "Starbucks"). Never run destructive scripts against it casually; the seeded `dana@roastwell.co` login does not exist there.
- **Device identity in every webhook comes from the authenticated MQTT `username`**, never `clientid` (client-supplied, spoofable). Prefer the `x-device-id` header, fall back to a `clientid` body field — mirror `app/api/mqtt/heartbeat/route.ts:43-54` exactly.
- **Every EMQX action must be type "HTTP Server" / "Webhook"** (it asks for a URL). "Republish" (asks for a Topic) forwards MQTT→MQTT and silently never reaches the cloud. This has broken production twice.
- **Money never moves for `billing: "included"`.** Credit settle/release stays behind `lib/trigger-ack.ts`.
- **Tests live in `lib/*.test.ts` as pure-function tests** — there are no route integration tests in this repo (45 test files, 438 tests). Put decision logic in pure helpers in `lib/` and keep routes thin, so the logic is testable in the established style.
- **Gates.** Cloud: `npm test`, `npx tsc --noEmit`, `npm run build`. Firmware: `make test`, `idf.py build`.
- **Command types are NOT widened.** Reuse the existing `config-changed` and `firmware-update` values in `lib/db/schema.ts:379`. They already have admin labels (`app/(tenant)/tenant/stores/[storeId]/[deviceId]/page.tsx:33-34`) and the manual "Update firmware" button (`components/devices/command-bar.tsx:11`) converges on the same seam. No migration for this.
- **A `config-changed` / `firmware-update` row stores `payload: null`.** The payload is generated at publish time and never persisted: a stored config replayed 60 s later would carry expired 300 s presigned R2 URLs.

## Deviations from the spec (deliberate, discovered while planning)

The spec at `docs/superpowers/specs/2026-07-29-mqtt-only-device-transport-design.md` is followed except for three points, all corrections rather than scope changes:

1. **Command type names.** Spec says `type: "config"` / `type: "ota"`. This plan reuses the existing `config-changed` / `firmware-update` types — zero enum churn, existing admin labels keep working, and the manual firmware-update button lands on the same publish seam.
2. **Payload persistence.** The spec says both new command types are persisted as rows "like every other command", which reads as storing the payload. Rows are persisted; **payloads are not**. Republish regenerates instead of replaying, because presigned URLs expire in 300 s (config) / 600 s (OTA manifest).
3. **Fragment reassembly is required on the device.** Measured config payload is **5,303 bytes** for the production org, while esp-mqtt's receive buffer is the IDF default **1,024 bytes** (no `CONFIG_MQTT_BUFFER_SIZE` override in `sdkconfig`). Inbound config therefore arrives as ~6 fragments and must be reassembled into PSRAM. Raising the esp-mqtt buffer instead is rejected: it lives in internal DRAM, where free space runs 120–170 KB and is already the system's tightest resource.

---

# PHASE A — Cloud, additive only

Nothing is deleted in this phase. Existing firmware keeps working throughout, and every task is independently revertible.

---

### Task A1: `mqttWebhookPing` table and channel-health helper

Gives every MQTT webhook a last-heard timestamp so a misconfigured EMQX rule is visible at a glance instead of failing silently.

**Files:**
- Modify: `lib/db/schema.ts` (add table after `firmwareRelease`, around line 423)
- Create: `lib/mqtt-ping.ts`
- Create: `lib/mqtt-ping.test.ts`
- Create: migration via `npm run db:generate`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `MQTT_CHANNELS: readonly ["ack", "heartbeat", "presence", "config-request"]`
  - `type MqttChannel = (typeof MQTT_CHANNELS)[number]`
  - `channelHealth(lastAt: Date | null, now: Date, staleMinutes?: number): "never" | "stale" | "ok"`
  - `async recordWebhookPing(channel: MqttChannel, deviceId: string | null): Promise<void>` — fail-open, never throws
  - `async getWebhookPings(): Promise<{ channel: string; lastAt: Date | null; lastDeviceId: string | null }[]>`

- [ ] **Step 1: Write the failing test**

Create `lib/mqtt-ping.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { MQTT_CHANNELS, channelHealth } from "./mqtt-ping";

describe("MQTT_CHANNELS", () => {
  it("covers every upstream webhook channel", () => {
    expect([...MQTT_CHANNELS]).toEqual(["ack", "heartbeat", "presence", "config-request"]);
  });
});

describe("channelHealth", () => {
  const now = new Date("2026-07-29T12:00:00Z");

  it("reports never for a channel that has never been heard", () => {
    expect(channelHealth(null, now)).toBe("never");
  });

  it("reports ok inside the stale window", () => {
    expect(channelHealth(new Date("2026-07-29T11:50:00Z"), now)).toBe("ok");
  });

  it("reports stale past the window", () => {
    expect(channelHealth(new Date("2026-07-29T11:00:00Z"), now)).toBe("stale");
  });

  it("treats the boundary as ok, not stale", () => {
    // 20 min default: exactly 20 min old is still ok, one ms older is stale.
    expect(channelHealth(new Date("2026-07-29T11:40:00Z"), now)).toBe("ok");
    expect(channelHealth(new Date("2026-07-29T11:39:59.999Z"), now)).toBe("stale");
  });

  it("honours a custom window", () => {
    expect(channelHealth(new Date("2026-07-29T11:55:00Z"), now, 2)).toBe("stale");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run lib/mqtt-ping.test.ts`
Expected: FAIL — `Failed to resolve import "./mqtt-ping"`

- [ ] **Step 3: Add the table to the schema**

In `lib/db/schema.ts`, immediately after the `firmwareRelease` table (ends line 423):

```ts
/**
 * Last-heard timestamp per MQTT webhook channel (four rows, forever). EMQX's
 * rules API returns 403 for namespaced keys, so a rule's action type cannot be
 * verified from code — a channel that has gone silent while its siblings keep
 * talking is how a "Republish instead of HTTP Server" misconfiguration is
 * diagnosed. Channel is the PK: every webhook upserts its own row.
 */
export const mqttWebhookPing = pgTable("mqtt_webhook_ping", {
  channel: text("channel").primaryKey(),
  lastAt: timestamp("last_at").notNull(),
  lastDeviceId: text("last_device_id"),
});
```

- [ ] **Step 4: Write `lib/mqtt-ping.ts`**

```ts
// lib/mqtt-ping.ts
// Per-channel MQTT webhook liveness. Writes are fail-open: a telemetry row must
// never be the reason a device webhook returns non-200 (EMQX deactivates a rule
// whose endpoint keeps failing).

import { db } from "@/lib/db";
import { mqttWebhookPing } from "@/lib/db/schema";

export const MQTT_CHANNELS = ["ack", "heartbeat", "presence", "config-request"] as const;
export type MqttChannel = (typeof MQTT_CHANNELS)[number];

/** Default staleness window: the device heartbeat is every 5 min, so 20 min is
 *  four missed beats — long enough to avoid false alarms, short enough to catch
 *  a misconfigured rule the same session it was created. */
export const STALE_MINUTES = 20;

export function channelHealth(
  lastAt: Date | null,
  now: Date,
  staleMinutes = STALE_MINUTES,
): "never" | "stale" | "ok" {
  if (!lastAt) return "never";
  return now.getTime() - lastAt.getTime() > staleMinutes * 60_000 ? "stale" : "ok";
}

/** Record that `channel` just received a webhook. Never throws. */
export async function recordWebhookPing(
  channel: MqttChannel,
  deviceId: string | null,
): Promise<void> {
  try {
    await db
      .insert(mqttWebhookPing)
      .values({ channel, lastAt: new Date(), lastDeviceId: deviceId })
      .onConflictDoUpdate({
        target: mqttWebhookPing.channel,
        set: { lastAt: new Date(), lastDeviceId: deviceId },
      });
  } catch (err) {
    console.error("[mqtt-ping] upsert failed", { channel, err });
  }
}

export async function getWebhookPings(): Promise<
  { channel: string; lastAt: Date | null; lastDeviceId: string | null }[]
> {
  return db.select().from(mqttWebhookPing);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run lib/mqtt-ping.test.ts`
Expected: PASS (6 tests)

- [ ] **Step 6: Generate the migration**

Run: `npm run db:generate`

Then open the generated `.sql` under `drizzle/` and **strip it down to only the `mqtt_webhook_ping` table creation**. Drizzle's snapshot in this repo has known drift and emits spurious FK churn; shipping that churn is a hazard. The file should contain only:

```sql
CREATE TABLE "mqtt_webhook_ping" (
	"channel" text PRIMARY KEY NOT NULL,
	"last_at" timestamp NOT NULL,
	"last_device_id" text
);
```

- [ ] **Step 7: Wire the three existing webhooks**

In each of `app/api/mqtt/ack/route.ts`, `app/api/mqtt/heartbeat/route.ts`, `app/api/mqtt/presence/route.ts`, add the import:

```ts
import { recordWebhookPing } from "@/lib/mqtt-ping";
```

and call it immediately after the device identity is known and before the handler returns — in `heartbeat/route.ts` that is right after the `clientid.length === 0` guard (line 54):

```ts
  await recordWebhookPing("heartbeat", clientid);
```

Use `"ack"` / `"presence"` with that route's resolved device id in the other two.

- [ ] **Step 8: Run the full gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS, 444 tests (438 + 6)

- [ ] **Step 9: Commit**

```bash
git add lib/db/schema.ts lib/mqtt-ping.ts lib/mqtt-ping.test.ts drizzle/ app/api/mqtt/
git commit -m "feat(mqtt): per-channel webhook liveness table + health helper

EMQX's rules API 403s on namespaced keys, so a rule's action type cannot be
verified from code. Record a last-heard timestamp per webhook channel instead:
one silent channel beside three talking ones is how a Republish-instead-of-HTTP
misconfiguration gets caught. Writes are fail-open so telemetry can never make
a device webhook return non-200."
```

---

### Task A2: Config and OTA publish seams

The single place that builds and publishes a config or an OTA manifest. Every caller (webhook, branding change, firmware publish, heartbeat reconcile, republish) goes through here, so presigning always happens at publish time.

**Files:**
- Create: `lib/mqtt-push.ts`
- Create: `lib/mqtt-push.test.ts`

**Interfaces:**
- Consumes: `publishCommand`, `mqttEnabled` (`lib/mqtt.ts`); `getDeviceConfig` (`lib/data.ts`); `resolveEffectivePin` (`lib/pin-resolve.ts`); `presignedGetUrl` (`lib/storage.ts`); `latestFirmwareManifest` (`lib/firmware.ts`); `id` (`lib/ids.ts`).
- Produces:
  - `type PushTarget = { id: string; organizationId: string; storeId: string | null; pinMode: string | null; pinnedUrl: string | null }`
  - `async resolveDeviceConfigPayload(dev: PushTarget): Promise<DeviceConfigPayload | null>`
  - `async publishConfigCommand(dev: PushTarget, commandId: string): Promise<boolean>`
  - `async publishOtaCommand(deviceId: string, commandId: string): Promise<boolean>`
  - `republishKindFor(type: string): "config" | "ota" | "replay"`

- [ ] **Step 1: Write the failing test**

Create `lib/mqtt-push.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { republishKindFor } from "./mqtt-push";

describe("republishKindFor", () => {
  it("regenerates config commands instead of replaying them", () => {
    // A stored config payload replayed 60s later would carry expired 300s
    // presigned R2 URLs, so the heartbeat republish must rebuild it.
    expect(republishKindFor("config-changed")).toBe("config");
  });

  it("regenerates firmware-update commands", () => {
    // The manifest url is a 600s presigned GET — same expiry problem.
    expect(republishKindFor("firmware-update")).toBe("ota");
  });

  it("replays every other command type verbatim", () => {
    for (const t of ["trigger", "pin", "reboot", "refresh", "identify"]) {
      expect(republishKindFor(t)).toBe("replay");
    }
  });

  it("replays unknown types rather than dropping them", () => {
    expect(republishKindFor("something-new")).toBe("replay");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run lib/mqtt-push.test.ts`
Expected: FAIL — `Failed to resolve import "./mqtt-push"`

- [ ] **Step 3: Write `lib/mqtt-push.ts`**

```ts
// lib/mqtt-push.ts
// The only place a config or OTA manifest is put on the wire. Both payloads
// embed short-lived presigned R2 URLs (config images 300s, firmware binary
// 600s), so they are built at publish time and NEVER persisted on the command
// row — a replay minutes later would ship dead URLs.

import { desc, eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { firmwareRelease, store as storeTable, tenantSettings } from "@/lib/db/schema";
import { getDeviceConfig, type DeviceConfigPayload } from "@/lib/data";
import { resolveEffectivePin } from "@/lib/pin-resolve";
import { presignedGetUrl } from "@/lib/storage";
import { latestFirmwareManifest } from "@/lib/firmware";
import { publishCommand } from "@/lib/mqtt";

/** The device columns the push seam needs. Matches the shape the device row and
 *  the config route already select, so callers can pass a full device row. */
export type PushTarget = {
  id: string;
  organizationId: string;
  storeId: string | null;
  pinMode: string | null;
  pinnedUrl: string | null;
};

/**
 * Build the payload the device used to fetch over GET /api/device/config:
 * effective pin resolved server-side (device > store > tenant), images presigned
 * fresh. Returns null when the org has no resolvable config.
 */
export async function resolveDeviceConfigPayload(
  dev: PushTarget,
): Promise<DeviceConfigPayload | null> {
  const [storeRow, [ts]] = await Promise.all([
    dev.storeId
      ? db
          .select({ pinMode: storeTable.pinMode, pinnedUrl: storeTable.pinnedUrl })
          .from(storeTable)
          .where(eq(storeTable.id, dev.storeId))
          .limit(1)
          .then((r) => r[0] ?? null)
      : Promise.resolve(null),
    db
      .select({ pinnedUrl: tenantSettings.pinnedUrl })
      .from(tenantSettings)
      .where(eq(tenantSettings.organizationId, dev.organizationId)),
  ]);
  const effective = resolveEffectivePin({
    device: { pinMode: dev.pinMode, pinnedUrl: dev.pinnedUrl },
    store: storeRow,
    tenant: { pinnedUrl: ts?.pinnedUrl ?? null },
  });
  // No If-None-Match: a push always carries the full config. 304 semantics
  // belonged to the HTTP route and have no meaning on a one-way publish.
  const { payload } = await getDeviceConfig(dev.organizationId, null, { url: effective.url });
  return payload;
}

/** Publish the device's full config on its cmd topic. False when disabled or on failure. */
export async function publishConfigCommand(dev: PushTarget, commandId: string): Promise<boolean> {
  const payload = await resolveDeviceConfigPayload(dev);
  if (!payload) return false;
  return publishCommand(dev.id, {
    commandId,
    type: "config-changed",
    action: null,
    payload,
  });
}

/** Publish the latest firmware manifest (fresh presigned binary URL). False when
 *  nothing is published, MQTT is disabled, or the publish failed. */
export async function publishOtaCommand(deviceId: string, commandId: string): Promise<boolean> {
  const [rel] = await db
    .select()
    .from(firmwareRelease)
    .orderBy(desc(firmwareRelease.createdAt))
    .limit(1);
  if (!rel) return false;
  const url = await presignedGetUrl(rel.r2Key, 600);
  return publishCommand(deviceId, {
    commandId,
    type: "firmware-update",
    action: null,
    payload: latestFirmwareManifest(rel, url),
  });
}

/**
 * How the heartbeat republish should resend a stale pending command. Payload-
 * carrying types must be rebuilt (their presigned URLs expire); everything else
 * is replayed from the stored row. Unknown types replay rather than drop, so a
 * future command type is delivered late instead of never.
 */
export function republishKindFor(type: string): "config" | "ota" | "replay" {
  if (type === "config-changed") return "config";
  if (type === "firmware-update") return "ota";
  return "replay";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run lib/mqtt-push.test.ts`
Expected: PASS (4 tests)

- [ ] **Step 5: Run the full gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS, 448 tests

- [ ] **Step 6: Commit**

```bash
git add lib/mqtt-push.ts lib/mqtt-push.test.ts
git commit -m "feat(mqtt): config + OTA publish seams, built fresh at publish time

One place builds and publishes a device config or firmware manifest. Both embed
short-lived presigned R2 URLs (300s images, 600s binary), so the payload is
generated per publish and never stored on the command row. republishKindFor
encodes that: config-changed and firmware-update are rebuilt on republish,
everything else replays verbatim."
```

---

### Task A3: `cfg/get` payload parser

**Files:**
- Modify: `lib/mqtt.ts` (add after `parsePresencePayload`, line 221)
- Modify: `lib/mqtt.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `parseConfigRequestPayload(raw: unknown): { deviceId: string } | null` — reads a `clientid` field; returns null for anything else. The header path (`x-device-id`) is handled by the route, matching the heartbeat route's precedence.

- [ ] **Step 1: Write the failing test**

Append to `lib/mqtt.test.ts`:

```ts
describe("parseConfigRequestPayload", () => {
  it("accepts a body carrying the authenticated username as clientid", () => {
    expect(parseConfigRequestPayload({ clientid: "dev_abc" })).toEqual({ deviceId: "dev_abc" });
  });

  it("rejects a missing clientid", () => {
    expect(parseConfigRequestPayload({})).toBeNull();
  });

  it("rejects an empty clientid", () => {
    expect(parseConfigRequestPayload({ clientid: "" })).toBeNull();
  });

  it("rejects a non-string clientid", () => {
    expect(parseConfigRequestPayload({ clientid: 42 })).toBeNull();
  });

  it("rejects non-objects", () => {
    expect(parseConfigRequestPayload(null)).toBeNull();
    expect(parseConfigRequestPayload("dev_abc")).toBeNull();
  });
});
```

Add `parseConfigRequestPayload` to the existing import from `./mqtt` at the top of the file.

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run lib/mqtt.test.ts`
Expected: FAIL — `parseConfigRequestPayload is not a function`

- [ ] **Step 3: Implement it**

Append to `lib/mqtt.ts`:

```ts
/** The device's "send me my config" request. Body identity only — the route
 *  prefers the x-device-id header, exactly as the heartbeat route does. */
export function parseConfigRequestPayload(raw: unknown): { deviceId: string } | null {
  if (!isObject(raw)) return null;
  const clientid = raw.clientid;
  if (typeof clientid !== "string" || clientid.length === 0) return null;
  return { deviceId: clientid };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run lib/mqtt.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add lib/mqtt.ts lib/mqtt.test.ts
git commit -m "feat(mqtt): parse the cfg/get webhook payload"
```

---

### Task A4: `POST /api/mqtt/config-request` webhook

**Files:**
- Create: `app/api/mqtt/config-request/route.ts`

**Interfaces:**
- Consumes: `mqttEnabled`, `verifyWebhookSecret`, `parseConfigRequestPayload` (A3); `publishConfigCommand` (A2); `recordWebhookPing` (A1).
- Produces: the HTTP surface EMQX's fifth rule targets. Inserts a `config-changed` row with `payload: null` and publishes the config.

- [ ] **Step 1: Write the route**

```ts
// POST /api/mqtt/config-request — EMQX webhook fired when a device publishes to
// d/{id}/cfg/get (once per boot/reconnect). The cloud answers by publishing the
// device's full, freshly-presigned config on its cmd topic. This replaces
// GET /api/device/config: the device asks over MQTT and never over HTTP.

import { NextResponse } from "next/server";
import { eq, sql } from "drizzle-orm";
import { db } from "@/lib/db";
import { device as deviceTable, deviceCommand } from "@/lib/db/schema";
import { mqttEnabled, verifyWebhookSecret, parseConfigRequestPayload } from "@/lib/mqtt";
import { publishConfigCommand } from "@/lib/mqtt-push";
import { recordWebhookPing } from "@/lib/mqtt-ping";
import { id as genId } from "@/lib/ids";

export const runtime = "nodejs";

export async function POST(req: Request) {
  if (!mqttEnabled()) return NextResponse.json({ error: "MQTT disabled" }, { status: 503 });
  if (!verifyWebhookSecret(req)) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  // Text first so a misconfigured rule body template can be logged verbatim.
  const bodyText = await req.text();
  let raw: unknown;
  try {
    raw = JSON.parse(bodyText);
  } catch {
    console.error("[mqtt/config-request] malformed body:", bodyText.slice(0, 300));
    return NextResponse.json({ error: "Malformed body" }, { status: 400 });
  }

  // Identity from the AUTHENTICATED username: the x-device-id header set by the
  // rule, else a clientid body field. Never a device-supplied field.
  const headerId = req.headers.get("x-device-id")?.trim();
  const parsed = parseConfigRequestPayload(raw);
  const deviceId = headerId && headerId.length > 0 ? headerId : parsed?.deviceId ?? "";
  if (deviceId.length === 0) {
    console.error("[mqtt/config-request] missing device id:", bodyText.slice(0, 300));
    return NextResponse.json({ error: "Missing device id" }, { status: 400 });
  }

  const [dev] = await db
    .select({
      id: deviceTable.id,
      organizationId: deviceTable.organizationId,
      storeId: deviceTable.storeId,
      pinMode: deviceTable.pinMode,
      pinnedUrl: deviceTable.pinnedUrl,
    })
    .from(deviceTable)
    .where(eq(deviceTable.id, deviceId))
    .limit(1);
  if (!dev) return NextResponse.json({ error: "Unknown device" }, { status: 404 });

  await recordWebhookPing("config-request", dev.id);

  // A request also proves liveness — the device just published to the broker.
  const now = new Date();
  // Atomic in-DB decision: never resurrect a paused device, even under
  // concurrent writes (no stale JS-side status read driving this write).
  await db
    .update(deviceTable)
    .set({
      lastSeenAt: now,
      status: sql`CASE WHEN ${deviceTable.status} = 'paused' THEN ${deviceTable.status} ELSE 'online' END`,
    })
    .where(eq(deviceTable.id, dev.id));

  // Row first (payload NULL — never store the presigned config), then publish.
  // If the publish fails the row stays pending and the heartbeat republish
  // rebuilds it, so a lost answer self-heals within one heartbeat.
  const commandId = genId("cmd");
  await db.insert(deviceCommand).values({
    id: commandId,
    deviceId: dev.id,
    organizationId: dev.organizationId,
    type: "config-changed",
    status: "pending",
  });
  const published = await publishConfigCommand(dev, commandId);

  return NextResponse.json({ ok: true, published });
}
```

- [ ] **Step 2: Verify it compiles and the suite is green**

Run: `npx tsc --noEmit && npm test`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add app/api/mqtt/config-request/route.ts
git commit -m "feat(mqtt): cfg/get webhook answers with a freshly-presigned config

The device asks for its config once per boot by publishing to d/{id}/cfg/get;
this webhook resolves the effective pin, presigns images at that moment and
publishes the full config on the device's cmd topic. The command row carries a
NULL payload on purpose so a republish rebuilds rather than replaying expired
URLs."
```

---

### Task A5: Config changes push the config itself

**Files:**
- Modify: `lib/data.ts:1567-1605` (`enqueueConfigChangedForOrg`)

**Interfaces:**
- Consumes: `publishConfigCommand` (A2).
- Produces: same exported signature — `enqueueConfigChangedForOrg(organizationId, createdByUserId)`. Callers in `app/(tenant)/tenant/branding/actions.ts:197` and `app/(tenant)/tenant/device-settings/actions.ts:74` are unchanged.

- [ ] **Step 1: Replace the body**

Replace the function (and its doc comment) with:

```ts
/**
 * Push the CURRENT config to every device in an org after a branding or
 * device-settings change. The message carries the config itself: with the HTTP
 * device API gone there is no GET for a "config changed" nudge to trigger. One
 * pending row per device is recorded so delivery is observable and so the
 * heartbeat republish can rebuild it for a device that was powered off.
 */
export async function enqueueConfigChangedForOrg(
  organizationId: string,
  createdByUserId: string | null,
): Promise<void> {
  const devices = await db
    .select({
      id: deviceTable.id,
      organizationId: deviceTable.organizationId,
      storeId: deviceTable.storeId,
      pinMode: deviceTable.pinMode,
      pinnedUrl: deviceTable.pinnedUrl,
    })
    .from(deviceTable)
    .where(eq(deviceTable.organizationId, organizationId));
  if (devices.length === 0) return;

  const rows = devices.map((d) => ({
    id: genId("cmd"),
    deviceId: d.id,
    organizationId,
    type: "config-changed" as const,
    createdByUserId: createdByUserId ?? undefined,
  }));
  await db.insert(deviceCommand).values(rows);

  // payload stays NULL on the row; publishConfigCommand presigns per publish.
  await Promise.all(devices.map((d, i) => publishConfigCommand(d, rows[i].id)));
}
```

- [ ] **Step 2: Fix the imports**

In `lib/data.ts`, replace the `publishCommand` import with `publishConfigCommand` from `@/lib/mqtt-push` if `publishCommand` has no other use in the file. Check first:

Run: `grep -n "publishCommand" lib/data.ts`

Keep both imports if other call sites remain.

- [ ] **Step 3: Update the two callers' log messages**

Both `app/(tenant)/tenant/branding/actions.ts:199` and `app/(tenant)/tenant/device-settings/actions.ts:76` log "devices reconcile on next poll". There is no poll any more. Change both to:

```ts
    console.error("config push failed (devices reconcile on their next heartbeat)", err);
```

- [ ] **Step 4: Run the gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add lib/data.ts "app/(tenant)/tenant/branding/actions.ts" "app/(tenant)/tenant/device-settings/actions.ts"
git commit -m "feat(mqtt): a config change publishes the config, not a nudge

A 'go fetch your config' message only made sense while the device had an HTTP
GET to fall back on. Send the config itself instead — one round trip, and the
presigned image URLs are seconds old when they arrive."
```

---

### Task A6: Heartbeat reconciles OTA and rebuilds republished payloads

**Files:**
- Modify: `app/api/mqtt/heartbeat/route.ts:88-108`

**Interfaces:**
- Consumes: `republishKindFor`, `publishConfigCommand`, `publishOtaCommand` (A2); `firmwareUpdateAvailable` (`lib/device-status.ts:36`).
- Produces: no new exports. The heartbeat becomes the reconciliation seam: it catches an OTA the device missed while powered off, and rebuilds payload-carrying republishes.

- [ ] **Step 1: Widen the device select**

The `.returning(...)` on the device update (line 85) currently returns only `id`. The push seam needs the pin columns. Change it to:

```ts
    .returning({
      id: deviceTable.id,
      organizationId: deviceTable.organizationId,
      storeId: deviceTable.storeId,
      pinMode: deviceTable.pinMode,
      pinnedUrl: deviceTable.pinnedUrl,
    });
```

(The reconcile below reads the running version from the heartbeat payload
(`hb.version`), not from the row, so `firmwareVersion` is not selected.)

- [ ] **Step 2: Replace the republish loop**

Replace lines 106-108 (`for (const cmd of stale) { ... }`) with:

```ts
  for (const cmd of stale) {
    switch (republishKindFor(cmd.type)) {
      // Payload-carrying commands are REBUILT, never replayed: the config's
      // image URLs are presigned for 300s and the firmware binary's for 600s,
      // so a stored payload is already dead by the time a republish fires.
      case "config":
        await publishConfigCommand(dev, cmd.id);
        break;
      case "ota":
        await publishOtaCommand(dev.id, cmd.id);
        break;
      case "replay":
        await publishCommand(dev.id, {
          commandId: cmd.id,
          type: cmd.type,
          action: cmd.action,
          payload: cmd.payload,
        });
        break;
    }
  }
```

- [ ] **Step 3: Add the OTA reconcile**

After the republish loop and before the final `return`:

```ts
  // OTA reconcile: the hb already reports the running version, so the cloud can
  // notice a device that came back from being powered off during a firmware
  // publish and hand it the manifest now. Only when nothing OTA-ish is already
  // pending, so a device that is mid-download isn't nudged again.
  let otaQueued = false;
  if (hb.version) {
    const [rel] = await db
      .select({ version: firmwareRelease.version })
      .from(firmwareRelease)
      .orderBy(desc(firmwareRelease.createdAt))
      .limit(1);
    if (firmwareUpdateAvailable(hb.version, rel?.version ?? null)) {
      const [pendingOta] = await db
        .select({ id: deviceCommand.id })
        .from(deviceCommand)
        .where(
          and(
            eq(deviceCommand.deviceId, dev.id),
            eq(deviceCommand.type, "firmware-update"),
            // "In flight" per this codebase's idiom (see lib/credit-holds.ts and
            // the ack route) — the still-live HTTP poll marks commands delivered.
            inArray(deviceCommand.status, ["pending", "delivered"]),
            // A row outside the republish window has already been given up on
            // by the loop above (nothing else ever expires a non-trigger
            // command), so it must not block a fresh OTA nudge forever —
            // only a row still young enough to be republished counts as
            // "in flight".
            gt(deviceCommand.createdAt, new Date(now.getTime() - REPUBLISH_UNTIL_MS)),
          ),
        )
        .limit(1);
      if (!pendingOta) {
        const commandId = genId("cmd");
        await db.insert(deviceCommand).values({
          id: commandId,
          deviceId: dev.id,
          organizationId: dev.organizationId,
          type: "firmware-update",
          status: "pending",
        });
        otaQueued = await publishOtaCommand(dev.id, commandId);
      }
    }
  }

  return NextResponse.json({ ok: true, republished: stale.length, otaQueued });
```

Delete the old `return NextResponse.json({ ok: true, republished: stale.length });`.

- [ ] **Step 4: Fix the imports**

```ts
import { and, desc, eq, gt, inArray, lt, sql } from "drizzle-orm";
import { device as deviceTable, deviceCommand, firmwareRelease } from "@/lib/db/schema";
import { mqttEnabled, verifyWebhookSecret, parseHeartbeatPayload, publishCommand } from "@/lib/mqtt";
import { republishKindFor, publishConfigCommand, publishOtaCommand } from "@/lib/mqtt-push";
import { firmwareUpdateAvailable } from "@/lib/device-status";
import { recordWebhookPing } from "@/lib/mqtt-ping";
import { id as genId } from "@/lib/ids";
```

- [ ] **Step 5: Run the gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/api/mqtt/heartbeat/route.ts
git commit -m "feat(mqtt): heartbeat rebuilds stale payloads and reconciles OTA

Two changes to the heartbeat webhook, both consequences of losing the HTTP
backstop. Republish now rebuilds config-changed and firmware-update instead of
replaying stored payloads whose presigned URLs have expired. And because the hb
already reports the running firmware version, a device that was powered off
during a firmware publish is handed the manifest the moment it checks in."
```

---

### Task A7: OTA publish + manual firmware-update land on the same seam

**Files:**
- Modify: `lib/db/publish-firmware.ts`
- Modify: whichever server action the `firmware-update` button in `components/devices/command-bar.tsx:11` posts to (find it in step 1)

- [ ] **Step 1: Find the manual command action**

Run: `grep -rn "firmware-update\|COMMAND_TYPES\|isValidCommandType" --include=*.ts --include=*.tsx app lib components | grep -v test`

Identify the server action that inserts a `deviceCommand` row for a manual command. Read it before editing.

- [ ] **Step 2: Publish the manifest from the manual path**

In that action, after the row insert, when `type === "firmware-update"`, publish through the seam instead of relying on the device to fetch a manifest over HTTP:

```ts
  if (type === "firmware-update") {
    await publishOtaCommand(deviceId, commandId);
  } else {
    await publishCommand(deviceId, { commandId, type, action: null, payload: null });
  }
```

Import `publishOtaCommand` from `@/lib/mqtt-push`.

- [ ] **Step 3: Publish to the fleet when a release is published**

In `lib/db/publish-firmware.ts`, after the `firmwareRelease` row is inserted, enqueue and publish a `firmware-update` for every claimed device. Read the file first, then add:

```ts
  // Every claimed device across all orgs — firmware releases are platform-wide.
  const targets = await db
    .select({ id: device.id, organizationId: device.organizationId })
    .from(device)
    .where(isNotNull(device.claimedAt));
  for (const t of targets) {
    const commandId = genId("cmd");
    await db.insert(deviceCommand).values({
      id: commandId,
      deviceId: t.id,
      organizationId: t.organizationId,
      type: "firmware-update",
      status: "pending",
    });
    const ok = await publishOtaCommand(t.id, commandId);
    console.log(`  ${t.id}: ${ok ? "published" : "queued (offline; hb will retry)"}`);
  }
```

- [ ] **Step 4: Run the gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add lib/db/publish-firmware.ts app lib
git commit -m "feat(mqtt): publish the OTA manifest instead of expecting a fetch

Publishing a release now hands every claimed device a manifest with a fresh
presigned binary URL, and the admin's manual Update-firmware button goes through
the same seam. Offline devices keep a pending row and get it on their next
heartbeat."
```

---

### Task A7b: One fleet-push seam for both publishing paths

Added during execution. Task A7's discovery step found a **second** firmware-publishing path that planning missed: `publishFirmware` in `lib/actions/firmware.ts` (the `/admin/firmware` upload UI) inserts the `firmwareRelease` row exactly like the CLI script but never pushes to the fleet.

This is not a correctness hole — the heartbeat OTA reconcile (Task A6) picks up any new release within one heartbeat interval (5 minutes, `HB_EVERY_MS` on the device), so online devices still converge. But an admin publishing through the UI reasonably expects the same "push now" behavior the CLI has, and leaving the claimed-device filter and the NULL-payload rule duplicated in two places contradicts the single-seam principle Task A7 exists to establish.

**Files:**
- Modify: `lib/mqtt-push.ts` (add the shared helper)
- Modify: `lib/db/publish-firmware.ts` (call it instead of its own loop)
- Modify: `lib/actions/firmware.ts` (call it after the release insert succeeds)

**Interfaces:**
- Consumes: `publishOtaCommand` (Task A2).
- Produces: `async pushFirmwareToFleet(): Promise<{ published: number; queued: number }>` — selects every claimed device across all orgs, inserts a `pending` `firmware-update` row per device with a NULL payload, publishes the manifest, and returns how many were delivered now versus left queued for the heartbeat to retry.

- [ ] **Step 1: Add the helper to `lib/mqtt-push.ts`**

```ts
/**
 * Hand every claimed device the current firmware manifest. Both publishing
 * paths — the CLI script and the /admin/firmware upload action — call this right
 * after their firmwareRelease insert, so the claimed-device filter and the
 * NULL-payload rule live in exactly one place. A device that is offline keeps a
 * pending row and gets the manifest, freshly presigned, on its next heartbeat.
 */
export async function pushFirmwareToFleet(): Promise<{ published: number; queued: number }> {
  const targets = await db
    .select({ id: device.id, organizationId: device.organizationId })
    .from(device)
    .where(isNotNull(device.claimedAt));
  let published = 0;
  for (const t of targets) {
    const commandId = genId("cmd");
    await db.insert(deviceCommand).values({
      id: commandId,
      deviceId: t.id,
      organizationId: t.organizationId,
      type: "firmware-update",
      status: "pending",
    });
    if (await publishOtaCommand(t.id, commandId)) published++;
  }
  return { published, queued: targets.length - published };
}
```

Add `device`, `deviceCommand` to the schema import, `isNotNull` to the drizzle import, and the repo's id generator (`id` from `@/lib/ids`, aliased as `genId` to match this file's existing style if it already imports one).

- [ ] **Step 2: Point the CLI script at it**

Replace the per-device loop Task A7 added to `lib/db/publish-firmware.ts` with a single call, keeping a summary log:

```ts
  const { published, queued } = await pushFirmwareToFleet();
  console.log(`  fleet: ${published} published, ${queued} queued (offline; hb will retry)`);
```

Delete the header comment A7 added about the second path being un-wired — it is wired now.

- [ ] **Step 3: Point the admin action at it**

In `lib/actions/firmware.ts`, call `pushFirmwareToFleet()` after the `firmwareRelease` insert succeeds. Wrap it so a push failure cannot fail the upload — the release row is the source of record and the heartbeat reconcile is the backstop:

```ts
  try {
    await pushFirmwareToFleet();
  } catch (err) {
    console.error("fleet OTA push failed (devices reconcile on their next heartbeat)", err);
  }
```

- [ ] **Step 4: Run the gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS, 453 tests

Do NOT execute `lib/db/publish-firmware.ts` — it would publish to the real fleet.

- [ ] **Step 5: Commit**

```bash
git add lib/mqtt-push.ts lib/db/publish-firmware.ts lib/actions/firmware.ts
git commit -m "refactor(mqtt): one fleet-push seam for both firmware publish paths

The /admin/firmware upload action created a release row but never pushed it, so
it relied entirely on the heartbeat reconcile while the CLI pushed immediately.
Extract the fan-out into pushFirmwareToFleet and call it from both, so the
claimed-device filter and the NULL-payload rule have one home."
```

---

### Task A8: Admin MQTT health card

**Files:**
- Modify: the platform-admin page that best fits a transport-health panel — find it in step 1
- Create: `components/admin/mqtt-health-card.tsx`

- [ ] **Step 1: Pick the host page**

Run: `ls "app/(admin)/admin"` and read the closest existing health/observability page. Follow its layout primitives exactly: `PageHeader` / `SectionHeader` / `PageSection`, `space-y-3` heading→body, `gap-4` metric grids. Do not re-pad the page — the shell owns the container.

- [ ] **Step 2: Write the card**

```tsx
// components/admin/mqtt-health-card.tsx
// Per-channel MQTT webhook liveness. EMQX's rules API 403s on namespaced keys,
// so a rule's action type can't be checked from code — one silent channel beside
// its talking siblings is the tell for a Republish-instead-of-HTTP-Server rule.

import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { MQTT_CHANNELS, channelHealth, getWebhookPings } from "@/lib/mqtt-ping";

const LABEL: Record<string, string> = {
  ack: "Command acks",
  heartbeat: "Heartbeats",
  presence: "Presence",
  "config-request": "Config requests",
};

export async function MqttHealthCard() {
  const pings = await getWebhookPings();
  const byChannel = new Map(pings.map((p) => [p.channel, p]));
  const now = new Date();

  return (
    <Card>
      <CardHeader>
        <CardTitle>MQTT channels</CardTitle>
      </CardHeader>
      <CardContent className="space-y-3">
        {MQTT_CHANNELS.map((ch) => {
          const p = byChannel.get(ch) ?? null;
          const health = channelHealth(p?.lastAt ?? null, now);
          return (
            <div key={ch} className="flex items-center justify-between gap-4 text-sm">
              <span>{LABEL[ch]}</span>
              <div className="flex items-center gap-3">
                <span className="text-muted-foreground">
                  {p?.lastAt
                    ? `${p.lastAt.toISOString().slice(0, 16).replace("T", " ")} UTC`
                    : "never"}
                </span>
                <Badge variant={health === "ok" ? "secondary" : "destructive"}>
                  {health === "ok" ? "live" : health === "stale" ? "silent" : "never seen"}
                </Badge>
              </div>
            </div>
          );
        })}
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 3: Mount it on the host page**

Render `<MqttHealthCard />` in the page picked in step 1.

- [ ] **Step 4: Run the gate**

Run: `npm test && npx tsc --noEmit && npm run build`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add components/admin/mqtt-health-card.tsx "app/(admin)"
git commit -m "feat(admin): MQTT channel health card

Surfaces each webhook channel's last-heard time. With the HTTP transport gone,
a silently misconfigured EMQX rule takes a whole subsystem down — this makes it
a glance instead of an investigation."
```

---

### Task A9: EMQX fifth rule + runbook

**Files:**
- Modify: `docs/runbooks/emqx-setup.md`

- [ ] **Step 1: Read the existing runbook**

Read `docs/runbooks/emqx-setup.md` in full and match its structure for the new rule section.

- [ ] **Step 2: Document the rule**

Add a section for the `cfg/get` rule, mirroring the heartbeat rule's shape:

````markdown
### Rule 4 — config requests (`d/+/cfg/get` → `/api/mqtt/config-request`)

The device publishes an empty `{}` to `d/{deviceId}/cfg/get` once per boot to ask
for its config. **The action type MUST be "HTTP Server" / "Webhook"** — the one
that asks for a URL. "Republish" asks for a Topic, forwards MQTT→MQTT and never
reaches the cloud; that mistake has silently broken this system twice.

- **SQL:** `SELECT username as clientid FROM "d/+/cfg/get"`
- **Action:** HTTP Server → `POST https://ditto-admin-brown.vercel.app/api/mqtt/config-request`
- **Headers:**
  - `x-emqx-webhook-secret: $EMQX_WEBHOOK_SECRET`
  - `x-device-id: ${username}`
- **Body:** `{"clientid":"${username}"}`

`username` is the authenticated identity; `clientid` is client-supplied and
spoofable. Never key this rule on `${clientid}`.

**Verify after creating it:** power-cycle a device and check the admin MQTT
channel card — "Config requests" must flip to *live*. If it stays *never seen*
while heartbeats are live, the action type is wrong.
````

- [ ] **Step 3: Grant the `cfg/get` publish ACL (operator step — discovered during execution)**

Planning missed this and it is fail-closed. The EMQX authorization rule set (runbook §3b) granted only `d/${username}/cmd` (subscribe), `ack` (publish) and `hb` (publish), followed by a catch-all `deny #`. Without an allow rule for `cfg/get`, **the broker rejects the device's config request before the webhook rule can ever fire** — and the symptom is indistinguishable from the Republish-vs-HTTP-Server mistake, so an operator would debug the wrong thing. The runbook's §3b curl now includes:

```json
{"topic":"d/${username}/cfg/get","permission":"allow","action":"publish"}
```

placed **before** the catch-all deny (EMQX evaluates the array in order and stops at first match, so an allow after the deny is dead). Re-run §3b's curl against the live deployment.

- [ ] **Step 4: Create the webhook rule in the EMQX console**

A manual console step; follow the section just written. **Do this after Task A10's deploy**, not before: the rule's target URL must exist, and EMQX deactivates a rule whose endpoint keeps failing. Nothing publishes to `cfg/get` until the firmware of Phase B ships, so there is no rush and no window of lost messages either way.

- [ ] **Step 4: Commit**

```bash
git add docs/runbooks/emqx-setup.md
git commit -m "docs(runbook): the cfg/get webhook rule"
```

---

### Task A10: Deploy Phase A

- [ ] **Step 1: Full gate**

Run: `npm test && npx tsc --noEmit && npm run build`
Expected: all PASS

- [ ] **Step 2: Apply the migration to Neon**

Run: `npm run db:migrate`

- [ ] **Step 3: Deploy**

Run: `vercel --prod --yes`

If the project link is missing: `vercel link --yes --scope eren-altans-projects --project ditto-admin` first.

- [ ] **Step 4: Do Task A9's two operator steps, in this order**

Both need EMQX console/API access and cannot be automated from this repo:
1. Re-run the runbook §3b authorization curl so the `cfg/get` publish ACL exists.
2. Create the config-request webhook rule per runbook §4 — action type **HTTP Server**, never Republish.

The rule is created *after* the deploy so its target URL already resolves.

- [ ] **Step 5: Verify the additive phase changed nothing for the live device**

The device is still on 0.17.1 and still uses HTTP. Confirm it stays online and that the admin MQTT card shows ack/heartbeat/presence live. "Config requests" will read *never seen* until Phase B ships — that is expected here, because no firmware publishes to `cfg/get` yet.

Also load `/admin/health` and confirm the MQTT transport card renders real rows rather than "Channel telemetry is unavailable" — that string means the migration did not apply.

- [ ] **Step 6: Confirm and report**

Report the deployment URL and the four channel states. **Do not start Phase C.**

---

# PHASE B — Firmware (repo: `/home/meren/projects/ditto-firmware`)

Export the toolchain first in every shell: `. $HOME/.espressif/v5.5/esp-idf/export.sh`

Set the git identity before the first commit — it is unset globally on this box:
`git config user.name "Eren Altan" && git config user.email "erenaltan@gmail.com"`

---

### Task B1: Extract config-body parsing from the HTTP fetch

The JSON→`device_config_t` parse is the valuable half of `cloud_get_config`; only its transport is changing.

**Files:**
- Modify: `components/cloud/cloud.c` (config section, around line 287-360)
- Modify: `components/cloud/include/cloud.h`
- Modify: the cfg-harness test target (find it via `make test`)

**Interfaces:**
- Produces: `bool cloud_config_parse_body(const char *json, int len, device_config_t *out);` — true when the body parsed into a valid config. `cloud_get_config` keeps working by calling it (Phase B5 deletes the HTTP wrapper).

- [ ] **Step 1: Read the current implementation**

Read `components/cloud/cloud.c` lines 287-362 and the cfg-harness test files. Identify exactly where the HTTP response body becomes a parsed `device_config_t`.

- [ ] **Step 2: Write the failing harness test**

Add to the cfg-harness a test that feeds a minimal config JSON straight to the new parser:

```c
static void test_parse_body_minimal(void)
{
    static device_config_t cfg;
    const char *json = "{\"version\":\"v1\",\"brandColor\":\"#10A765\",\"wordmark\":\"Acme\","
                       "\"config\":{\"clockTimezone\":\"UTC\",\"qrTimeoutSeconds\":15,\"screens\":{}},"
                       "\"device\":{\"brightness\":80,\"sleep\":{\"enabled\":false,\"timeoutSeconds\":300}},"
                       "\"pin\":null}";
    assert(cloud_config_parse_body(json, (int)strlen(json), &cfg));
    assert(cfg.valid);
    assert(strcmp(cfg.version, "v1") == 0);
    assert(cfg.device.brightness == 80);
    printf("ok parse_body_minimal\n");
}
```

- [ ] **Step 3: Run it and watch it fail**

Run: `make test`
Expected: FAIL — implicit declaration of `cloud_config_parse_body`

- [ ] **Step 4: Extract the function**

Move the parse block out of `cloud_get_config` into:

```c
bool cloud_config_parse_body(const char *json, int len, device_config_t *out)
{
    // ... the exact parse body that used to live inline in cloud_get_config,
    // operating on (json, len) instead of the HTTP response buffer.
}
```

and have `cloud_get_config` call it on its response buffer. Declare it in `cloud.h` next to `cloud_get_config`.

- [ ] **Step 5: Run the tests**

Run: `make test`
Expected: PASS, including the new case

- [ ] **Step 6: Build**

Run: `idf.py build`
Expected: success

- [ ] **Step 7: Commit**

```bash
git add components/cloud/
git commit -m "refactor(cloud): split config JSON parsing from the HTTP fetch

Only the transport is changing; the parse is the valuable part. Pulling it out
as cloud_config_parse_body lets an MQTT-delivered config reuse it verbatim."
```

---

### Task B2: Reassemble large inbound MQTT payloads into PSRAM

**This is the constraint that shapes the whole device side.** esp-mqtt's receive buffer is the IDF default 1024 bytes (no `CONFIG_MQTT_BUFFER_SIZE` in `sdkconfig`), and the measured config payload is **5,303 bytes** for the production org — so config arrives as ~6 `MQTT_EVENT_DATA` fragments. Raising the esp-mqtt buffer is not the fix: it lives in internal DRAM, the system's scarcest resource (120–170 KB free, and a single Settings visit already costs ~107 KB).

**Files:**
- Modify: `components/mqtt_ditto/mqtt_client.c`
- Modify: `components/mqtt_ditto/include/mqtt_client_ditto.h`

**Interfaces:**
- Produces:
  - `bool mqtt_config_pending(void);`
  - `bool mqtt_parse_pending_config(device_config_t *out);` — parses the staged JSON into `out` under the staging mutex and clears the pending flag. False if nothing staged or the parse failed.

- [ ] **Step 1: Add the staging buffer and reassembly**

In `mqtt_client.c`, add above `on_cmd_message`:

```c
// Inbound config payloads are ~5-20KB while esp-mqtt's receive buffer is 1KB,
// so MQTT_EVENT_DATA arrives fragmented. Reassemble into PSRAM (internal DRAM
// is the scarce resource) and let poll_task parse straight out of this slot
// under the mutex — the parse is milliseconds and inbound configs are rare.
#define STAGE_CAP (32 * 1024)
static char *s_stage;                  // PSRAM, lazily allocated
static int s_stage_len;
static volatile bool s_stage_ready;
static SemaphoreHandle_t s_stage_mux;

static bool stage_ensure(void)
{
    if (!s_stage_mux) s_stage_mux = xSemaphoreCreateMutex();
    if (!s_stage) s_stage = heap_caps_malloc(STAGE_CAP, MALLOC_CAP_SPIRAM);
    return s_stage && s_stage_mux;
}
```

- [ ] **Step 2: Route MQTT_EVENT_DATA through reassembly**

Replace the `MQTT_EVENT_DATA` case body (`mqtt_client.c:50-53`) with a call to a new handler:

```c
static void on_data(esp_mqtt_event_handle_t e)
{
    // Single-fragment message: parse in place, no staging needed.
    if (e->current_data_offset == 0 && e->data_len == e->total_data_len) {
        on_cmd_message(e->data, e->data_len);
        return;
    }
    if (!stage_ensure()) { ESP_LOGE(TAG, "stage alloc failed; dropping %d B", e->total_data_len); return; }
    if (e->total_data_len >= STAGE_CAP) {
        ESP_LOGE(TAG, "payload %d B exceeds stage cap %d", e->total_data_len, STAGE_CAP);
        return;
    }
    xSemaphoreTake(s_stage_mux, portMAX_DELAY);
    memcpy(s_stage + e->current_data_offset, e->data, e->data_len);
    bool complete = (e->current_data_offset + e->data_len >= e->total_data_len);
    if (complete) {
        s_stage_len = e->total_data_len;
        s_stage[s_stage_len] = '\0';
    }
    xSemaphoreGive(s_stage_mux);
    if (complete) on_cmd_message(s_stage, s_stage_len);
}
```

and in the event switch: `case MQTT_EVENT_DATA: on_data(e); break;`

- [ ] **Step 3: Flag a staged config instead of copying it into the queue**

`dev_command_t` is a small fixed struct copied by value through a queue — a config payload cannot ride in it. In `on_cmd_message`, after `command_parse_into` succeeds, mark the staged JSON as the pending config when the command is a config push:

```c
    if (strcmp(dc.type, "config-changed") == 0 && data == s_stage) {
        s_stage_ready = true;   // poll_task parses out of the staging slot
    }
```

Add the accessors:

```c
bool mqtt_config_pending(void) { return s_stage_ready; }

bool mqtt_parse_pending_config(device_config_t *out)
{
    if (!s_stage_ready || !s_stage || !s_stage_mux) return false;
    xSemaphoreTake(s_stage_mux, portMAX_DELAY);
    // The payload is the command envelope; the config lives under "payload".
    bool ok = cloud_config_parse_command(s_stage, s_stage_len, out);
    s_stage_ready = false;
    xSemaphoreGive(s_stage_mux);
    return ok;
}
```

- [ ] **Step 4: Add the envelope unwrapper to the cloud component**

The MQTT message is `{commandId, type, action, payload: <config>}`, so add next to `cloud_config_parse_body` in `cloud.c` (declared in `cloud.h`):

```c
// Unwrap a command envelope and parse its `payload` as a device config.
bool cloud_config_parse_command(const char *json, int len, device_config_t *out)
{
    cJSON *root = cJSON_ParseWithLength(json, len);
    if (!root) return false;
    cJSON *payload = cJSON_GetObjectItem(root, "payload");
    bool ok = false;
    if (cJSON_IsObject(payload)) {
        char *body = cJSON_PrintUnformatted(payload);
        if (body) {
            ok = cloud_config_parse_body(body, (int)strlen(body), out);
            cJSON_free(body);
        }
    }
    cJSON_Delete(root);
    return ok;
}
```

- [ ] **Step 5: Add a harness test for the envelope**

```c
static void test_parse_command_envelope(void)
{
    static device_config_t cfg;
    const char *msg = "{\"commandId\":\"cmd_1\",\"type\":\"config-changed\",\"action\":null,"
                      "\"payload\":{\"version\":\"v2\",\"brandColor\":\"#10A765\",\"wordmark\":\"Acme\","
                      "\"config\":{\"clockTimezone\":\"UTC\",\"qrTimeoutSeconds\":15,\"screens\":{}},"
                      "\"device\":{\"brightness\":70,\"sleep\":{\"enabled\":false,\"timeoutSeconds\":300}},"
                      "\"pin\":null}}";
    assert(cloud_config_parse_command(msg, (int)strlen(msg), &cfg));
    assert(cfg.valid);
    assert(strcmp(cfg.version, "v2") == 0);
    printf("ok parse_command_envelope\n");
}
```

- [ ] **Step 6: Run tests and build**

Run: `make test && idf.py build`
Expected: both PASS

- [ ] **Step 7: Commit**

```bash
git add components/mqtt_ditto/ components/cloud/
git commit -m "feat(mqtt): reassemble fragmented inbound payloads into PSRAM

esp-mqtt's receive buffer is 1KB and a real config payload measures 5.3KB, so
config arrives as ~6 fragments. Reassemble by offset into a 32KB PSRAM slot and
let poll_task parse out of it under a mutex. Growing the esp-mqtt buffer was
rejected: it lives in internal DRAM, which is the tight resource here."
```

---

### Task B3: Ask for config at boot over MQTT

**Files:**
- Modify: `components/mqtt_ditto/mqtt_client.c`, `components/mqtt_ditto/include/mqtt_client_ditto.h`, `components/mqtt_ditto/mqtt_topics.h`
- Modify: `main/app_state.c`

**Interfaces:**
- Produces: `void mqtt_publish_cfg_get(void);` — publishes `{}` to `d/{id}/cfg/get` at QoS 1. No-op when disconnected.

- [ ] **Step 1: Add the topic helper**

In `mqtt_topics.h`, alongside `mqtt_topic_cmd`:

```c
static inline void mqtt_topic_cfg_get(char *out, size_t cap, const char *device_id)
{
    snprintf(out, cap, "d/%s/cfg/get", device_id);
}
```

- [ ] **Step 2: Implement the publish**

In `mqtt_client.c`:

```c
void mqtt_publish_cfg_get(void)
{
    if (!s_client || !s_connected) return;
    char topic[80];
    mqtt_topic_cfg_get(topic, sizeof(topic), s_device_id);
    esp_mqtt_client_publish(s_client, topic, "{}", 2, 1, 0);
    ESP_LOGI(TAG, "cfg/get published");
}
```

- [ ] **Step 3: Call it on connect, with jitter**

In `app_state.c`'s `poll_task`, in the branch that fires on a fresh MQTT connection (where `!s_was_mqtt_up` currently sends the immediate hb), add the config request after the hb:

```c
                if (!s_was_mqtt_up) {
                    mqtt_publish_hb(/* ...existing args... */);
                    s_last_hb_ms = now_ms();
                    // Ask for config once per connect. Jitter so a store whose
                    // devices all power up together doesn't hit the cloud with
                    // simultaneous presign work.
                    vTaskDelay(pdMS_TO_TICKS(esp_random() % 3000));
                    mqtt_publish_cfg_get();
                }
```

Add `#include "esp_random.h"` to `app_state.c`.

- [ ] **Step 4: Build**

Run: `idf.py build`
Expected: success

- [ ] **Step 5: Commit**

```bash
git add components/mqtt_ditto/ main/app_state.c
git commit -m "feat(mqtt): request config over MQTT once per connect

The device asks for its config by publishing to d/{id}/cfg/get on every
connect — the only 'pull' left, and it happens once, not on a timer. A 0-3s
jitter keeps a whole store powering up at once from stampeding the cloud."
```

---

### Task B4: Apply MQTT-delivered config and OTA manifests

**Files:**
- Modify: `components/cloud/commands.c` (`dispatch_command`)
- Modify: `components/ota/ota.c`, `components/ota/include/ota.h`
- Modify: `main/app_state.c`

**Interfaces:**
- Consumes: `mqtt_config_pending`, `mqtt_parse_pending_config` (B2); `cloud_config_parse_command` (B2).
- Produces: `void ota_start_with_manifest(const fw_manifest_t *m);` replacing `ota_check_and_update`'s internal fetch.

- [ ] **Step 1: Read the OTA component**

Read `components/ota/ota.c` in full, in particular line 19 (`if (!cloud_get_firmware(&m)) return;`) and how `m` is used afterwards.

- [ ] **Step 2: Add the manifest-taking entry point**

Split `ota_check_and_update` so the download/verify/reboot half takes a manifest:

```c
void ota_start_with_manifest(const fw_manifest_t *m)
{
    if (!m || !m->version[0] || !m->url[0]) return;
    // ... the exact body that followed cloud_get_firmware, using *m
}
```

Declare it in `ota.h`.

- [ ] **Step 3: Parse the manifest out of the command envelope**

`fw_manifest_t` is small and fixed, so it fits in a queued command — but `dev_command_t` has no field for it. Rather than widen the struct, parse the manifest from the staged JSON the same way config is. Add to `components/devcfg/ota_manifest.c` (declared in `ota_manifest.h`):

```c
// Unwrap a command envelope and parse its `payload` as an OTA manifest.
bool ota_manifest_parse_command(const char *json, int len, fw_manifest_t *out)
{
    cJSON *root = cJSON_ParseWithLength(json, len);
    if (!root) return false;
    cJSON *payload = cJSON_GetObjectItem(root, "payload");
    bool ok = cJSON_IsObject(payload) && ota_manifest_parse(payload, out);
    cJSON_Delete(root);
    return ok;
}
```

Match the existing `ota_manifest_parse` signature found in step 1; if it takes a JSON string rather than a `cJSON *`, adapt accordingly.

- [ ] **Step 4: Add a staged-manifest accessor to mqtt_ditto**

Mirroring `mqtt_parse_pending_config`:

```c
bool mqtt_parse_pending_ota(fw_manifest_t *out)
{
    if (!s_ota_ready || !s_stage || !s_stage_mux) return false;
    xSemaphoreTake(s_stage_mux, portMAX_DELAY);
    bool ok = ota_manifest_parse_command(s_stage, s_stage_len, out);
    s_ota_ready = false;
    xSemaphoreGive(s_stage_mux);
    return ok;
}
```

Set `s_ota_ready = true` in `on_cmd_message` when `strcmp(dc.type, "firmware-update") == 0 && data == s_stage`, next to the config flag from B2.

- [ ] **Step 5: Dispatch both from `dispatch_command`**

In `commands.c`, replace the `config-changed` / `refresh` branch and the `firmware-update` branch:

```c
    } else if (strcmp(cmd->type, "config-changed") == 0 || strcmp(cmd->type, "refresh") == 0) {
        // The config now arrives IN the message (staged in PSRAM by mqtt_ditto);
        // the handler applies it instead of triggering an HTTP fetch.
        void (*cb)(void) = cloud_config_changed_handler();
        if (cb) cb();
        ack(id, true);
    } else if (strcmp(cmd->type, "pin") == 0) {
        void (*pcb)(void) = cloud_config_changed_handler();
        if (pcb) pcb();
        ack(id, true);
    } else if (strcmp(cmd->type, "firmware-update") == 0) {
        ack(id, true);   // ack BEFORE OTA (which reboots)
        void (*fcb)(void) = cloud_firmware_update_handler();
        if (fcb) fcb();
```

The handler bodies live in `app_state.c` — that is where the next step points them at the staged payloads.

- [ ] **Step 6: Rewire the handlers in `app_state.c`**

The `config_changed` handler currently sets `s_config_dirty`, and the poll loop then calls `cloud_get_config`. Replace that fetch (the `if (s_config_dirty)` block, around line 517) so it parses the staged payload instead:

```c
            if (s_config_dirty) {
                s_config_dirty = false;
                int next = s_cfg_live ^ 1;
                *s_cfg_buf[next] = *s_cfg_buf[s_cfg_live];   // seed from the live config
                bool ok = mqtt_parse_pending_config(s_cfg_buf[next]);
                if (ok && s_cfg_buf[next]->valid) {
                    // ... the existing apply block, unchanged:
                    // prefetch_and_evict_assets / ui_set_config / brightness /
                    // tz / countdown / s_cfg_live = next / mqtt start-stop /
                    // show_home or ui_render_state
                } else {
                    // Nothing staged, or it failed to parse. Ask again — the
                    // cloud answers cfg/get with a freshly presigned config.
                    mqtt_publish_cfg_get();
                }
            }
```

Note: the `mqtt_start`/`mqtt_stop` calls inside the apply block stay — a config that turns MQTT off must still be honoured.

For the firmware handler, replace the `ota_check_and_update(false)` call path:

```c
static void on_firmware_update(void)
{
    fw_manifest_t m;
    if (mqtt_parse_pending_ota(&m)) ota_start_with_manifest(&m);
    else ESP_LOGW(TAG, "firmware-update with no staged manifest");
}
```

and delete the periodic `s_ota_poll_ctr` / `OTA_CHECK_EVERY` block from `poll_task` — the cloud now decides when an OTA is due (heartbeat version reconcile), so the device no longer checks on a timer.

- [ ] **Step 7: Build**

Run: `idf.py build`
Expected: success

- [ ] **Step 8: Commit**

```bash
git add components/ main/app_state.c
git commit -m "feat(device): apply MQTT-delivered config and OTA manifests

Config and the firmware manifest now arrive in the command message. The config
handler parses the PSRAM-staged payload into the idle buffer on poll_task, so
the renderer still never sees a half-written config; a miss re-asks over
cfg/get. The periodic OTA check is gone — the cloud reconciles versions from
the heartbeat and pushes when the device is behind."
```

---

### Task B5: Delete the HTTP device transport

**Files:**
- Modify: `components/cloud/cloud.c`, `components/cloud/include/cloud.h`
- Modify: `components/cloud/commands.c`, `components/cloud/include/commands.h`
- Modify: `main/app_state.c`

- [ ] **Step 1: Delete the HTTP functions**

Remove from `cloud.c` + `cloud.h`: `cloud_get_commands`, `cloud_get_commands_to`, `cloud_ack_command`, `cloud_get_config`, `cloud_get_firmware`, and `cloud_post_document` (already dead — ingest was removed in the trigger-only pivot).

Keep: `cloud_claim_poll`, `cloud_fetch_asset`, `cloud_config_load_cached`, `cloud_config_parse_body`, `cloud_config_parse_command`, `cloud_last_asset_status`, `cloud_last_config_status`, `cloud_last_parse_ok`, and the handler setters/getters.

- [ ] **Step 2: Delete `commands_handle_body`**

Remove it from `commands.c` and `commands.h` — it existed only to parse an HTTP poll response.

- [ ] **Step 3: Collapse `poll_task`'s two paths into one**

In `app_state.c`, delete the `mqtt_up` branching around the command fetch (the `if (mqtt_up) status = 200; else if (s_boot_gate) ... cloud_get_commands_to ... else cloud_get_commands` block), the `if (!mqtt_up) commands_handle_body(body);` line, the `static char body[1024]` buffer, and `POLL_IDLE_MS`.

The loop keeps: boot-gate handling, input consumption, `net_is_connected` guard, the MQTT command drain, the config-apply block, heartbeat publishing, and `idle_wait_or_qr_expiry(MQTT_IDLE_MS)`.

Online-ness now follows the broker, so replace the old `status == 200` gate with `mqtt_is_connected()`:

```c
        if (mqtt_is_connected()) {
            ui_set_online(true);
            time_sync_start();
            // ... rssi, boot-gate release, config apply, heartbeat ...
            idle_wait_or_qr_expiry(MQTT_IDLE_MS);
        } else {
            ESP_LOGW(TAG, "mqtt down; waiting for reconnect (backoff=%dms)", backoff);
            ui_set_online(false);
            vTaskDelay(pdMS_TO_TICKS(backoff));
            backoff = backoff * 2 > POLL_BACKOFF_MAX ? POLL_BACKOFF_MAX : backoff * 2;
        }
```

Also drop the "HTTP will redeliver" log in the MQTT drain (line ~485) — there is no HTTP to redeliver.

- [ ] **Step 4: Bump the version**

Set the firmware version to `0.18.0` wherever `appcfg_fw_version` reads it (check `main/Kconfig.projbuild` and the project `CMakeLists.txt`).

- [ ] **Step 5: Run tests and build**

Run: `make test && idf.py build`
Expected: both PASS, with no warnings about unused statics

- [ ] **Step 6: Commit**

```bash
git add components/ main/
git commit -m "feat(device)!: delete the HTTP transport; MQTT only; bump 0.18.0

Removes command polling, HTTP ack, the config fetch, the firmware-manifest fetch
and the long-dead ingest post. poll_task loses its dual path: online-ness now
follows the broker connection and there is no fallback by design. HTTPS remains
only for the one-time claim, R2 asset fetches and the OTA binary download."
```

---

### Task B6: Flash and HIL

- [ ] **Step 1: Flash the device**

Run: `sg uucp -c 'idf.py -p /dev/ttyACM0 flash'`

(`idf.py monitor` traps the board in download mode on this box — use `python tools/serial-read.py /dev/ttyACM0 25` to read the console instead.)

- [ ] **Step 2: Verify the boot sequence on the console**

Expected, in order: SDIO slave identified → Wi-Fi from NVS → IP → `mqtt connected, subscribed d/<id>/cmd` → `cfg/get published` → a config applied and the idle screen rendered with the correct branding. There must be **no** `GET /commands` line anywhere.

- [ ] **Step 3: Verify the cloud side**

In the admin: the device shows online, and the MQTT channel card shows **all four** channels live — including "Config requests".

- [ ] **Step 4: HIL — config change**

Change the brand colour in `/tenant/branding` and save. The device must repaint within seconds without being touched.

- [ ] **Step 5: HIL — trigger**

Fire a trigger through `POST /api/v1/devices/{id}/trigger`. Confirm the QR appears, the command reaches `acked`, and a credit settles (org must be on the `credits` plan).

- [ ] **Step 6: HIL — power-cycle recovery**

Power the device off, change the brand colour while it is off, power it back on. The device must come up showing the **new** colour — proving the boot `cfg/get` catches a change missed while offline.

- [ ] **Step 7: HIL — broker outage**

Ban the device's MQTT user in the EMQX console. Expected: the device goes offline in the UI, the screen keeps showing its cached idle state, and no HTTP fallback traffic appears. Unban; the device must reconnect and re-request config on its own.

- [ ] **Step 8: Record the results and commit any fixes**

If a fix is needed, commit it with a message describing the HIL finding. Do not proceed to Phase B2 until steps 2-7 all pass.

---

# PHASE B2 — Publish the OTA and confirm convergence

### Task B7: Publish 0.18.0 and verify the fleet converged

- [ ] **Step 1: Publish the release**

In the cloud repo: `npx tsx lib/db/publish-firmware.ts` with the built `0.18.0` binary, per that script's usage. It now also enqueues and publishes a `firmware-update` to every claimed device.

- [ ] **Step 2: Confirm the device took it**

The device should download, verify, reboot and report `0.18.0` in its heartbeat. Confirm `firmwareVersion` is `0.18.0` in the admin.

- [ ] **Step 3: Gate check — this is the Phase C precondition**

Confirm **every device that must survive** reports 0.18.0. Only `dev_by5LqtjAFPEPkLNFRsz2u` (Printer b580) matters; the two `0.6.0-m6b` rows are being retired in Task C3.

**Do not start Phase C until this step is confirmed.** Phase C removes the config and OTA HTTP paths simultaneously; a device still on 0.17.1 at that point can only be recovered over USB.

---

# PHASE C — Cloud, subtractive

Precondition: Task B7 step 3 confirmed.

---

### Task C1: Delete the HTTP device routes

**Files:**
- Delete: `app/api/device/commands/route.ts`
- Delete: `app/api/device/commands/ack/route.ts`
- Delete: `app/api/device/config/route.ts`
- Delete: `app/api/device/firmware/route.ts`
- Delete: `lib/device-auth.ts`
- Modify: `lib/mqtt.ts` (comments and two now-misleading doc blocks)

- [ ] **Step 1: Confirm nothing else imports them**

Run: `grep -rn "device-auth\|api/device/commands\|api/device/config\|api/device/firmware" --include=*.ts --include=*.tsx app lib components | grep -v node_modules`

Expected: only the files being deleted. `app/api/device/claim/route.ts` must NOT appear (it is unauthenticated and stays).

- [ ] **Step 2: Delete them**

```bash
git rm app/api/device/commands/route.ts app/api/device/commands/ack/route.ts \
       app/api/device/config/route.ts app/api/device/firmware/route.ts lib/device-auth.ts
```

- [ ] **Step 3: Fix the stale comments in `lib/mqtt.ts`**

The module header (lines 2-5) says the transport "falls back to HTTP polling" — no longer true. Replace with:

```ts
// lib/mqtt.ts
// EMQX / MQTT device transport helpers. Pure and testable except publishCommand,
// which performs the single outbound HTTP publish. Everything gates on
// mqttEnabled(): MQTT is the ONLY device transport, so with the EMQX env group
// absent no device can be reached at all — there is deliberately no fallback.
```

`buildMqttConfigBlock`'s comment references `/api/device/config` (line 27) and `mqttConfigFingerprint`'s references the config ETag (lines 45-49). The device now receives the `mqtt` block inside the pushed config, and there is no 304 path, so:

```ts
/** The `mqtt` block inside the pushed device config, or null when disabled. The
 *  device authenticates to the broker with its own device key as the MQTT
 *  password (username = deviceId), so no secret is carried here. */
```

```ts
/** Broker identity (host+port) folded into the config version so a broker move
 *  is reflected in a device's next config. Per-device clientId/username are the
 *  stable device id and are excluded. */
```

- [ ] **Step 4: Run the gate**

Run: `npm test && npx tsc --noEmit && npm run build`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git commit -am "feat!: delete the HTTP device API

Command polling, HTTP ack, the config fetch and the firmware manifest are all
served over MQTT now, so the routes and lib/device-auth.ts (whose only callers
they were) are gone. The device key survives purely as the MQTT password.
/api/device/claim stays: bootstrap has no credential to authenticate with yet."
```

---

### Task C2: A trigger that cannot be published fails loudly

**Files:**
- Modify: `app/api/v1/devices/[deviceId]/trigger/route.ts:97-107`

- [ ] **Step 1: Replace the best-effort publish**

```ts
  // MQTT is the only transport, so a failed publish means the command will never
  // reach the device. A trigger delivered minutes later is worthless — the
  // customer is at the counter now — and leaving the row pending would make the
  // heartbeat republish show an unwanted QR later. Fail closed instead: mark it
  // failed, refund the reservation, and tell the caller.
  const published = await publishCommand(deviceId, {
    commandId,
    type: "trigger",
    action: v.action,
    payload: v.payload,
  });
  if (!published) {
    await db
      .update(deviceCommand)
      .set({ status: "failed", result: "publish_failed" })
      .where(eq(deviceCommand.id, commandId));
    await cancelTriggerReservation({
      organizationId: auth.organizationId,
      deviceId,
      commandId,
      cost,
      billing: reserved.billing,
      month: reserved.month,
    });
    await db
      .delete(apiIdempotency)
      .where(
        and(
          eq(apiIdempotency.key, idemKey),
          eq(apiIdempotency.organizationId, auth.organizationId),
        ),
      );
    return apiError(
      "transport_unavailable",
      "Could not reach the device transport. No credit was charged; retry.",
      503,
    );
  }

  return apiJson(body, 202);
```

The idempotency row is deleted so a retry can proceed — a 503 must not be replayed as a stored success.

- [ ] **Step 2: Update the OpenAPI spec**

Add the 503 `transport_unavailable` response to the trigger endpoint in `app/api/v1/openapi.json/route.ts`.

- [ ] **Step 3: Run the gate**

Run: `npm test && npx tsc --noEmit`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git commit -am "fix(trigger)!: fail closed when the MQTT publish fails

The route published best-effort because HTTP polling was the backstop; it isn't
any more. A publish failure now marks the command failed, cancels the credit
reservation, releases the idempotency key so a retry works, and returns 503
transport_unavailable — instead of a 202 for a trigger that will never arrive."
```

---

### Task C3: Retire the pre-MQTT device rows

**Files:**
- Create: `scripts/retire-legacy-devices.mts` (delete after running, or keep if the repo has a `scripts/` convention — check first)

- [ ] **Step 1: Confirm what will be deleted**

Write and run a read-only listing first:

```ts
import "@/lib/db/load-env";
import { db } from "@/lib/db";
import { device } from "@/lib/db/schema";
const rows = await db.select().from(device);
console.log(rows.map((r) => ({ id: r.id, name: r.name, fw: r.firmwareVersion, claimed: r.claimedAt })));
```

Expected: `dev_by5LqtjAFPEPkLNFRsz2u` on 0.18.0, plus `dev_Kb7fdQxNyvQdF1PwT2bOa` and `dev_0XSFnFN4-njdVpMDRXLlE` on `0.6.0-m6b`.

- [ ] **Step 2: Deprovision their MQTT credentials, then delete the rows**

Use the existing chokepoint rather than raw SQL so EMQX credentials are cleaned up too — find it with `grep -rn "deleteDevice" lib app --include=*.ts | grep -v test` and call that function for each legacy id.

- [ ] **Step 3: Verify**

Re-run the step 1 listing. Expected: one device row remains.

- [ ] **Step 4: Commit**

```bash
git add scripts/ 2>/dev/null; git commit -m "chore: retire the two pre-MQTT device rows

0.6.0-m6b cannot speak MQTT and had no path forward after the HTTP transport
was removed. One was never claimed." --allow-empty
```

---

### Task C4: Update the docs that now describe a dead transport

**Files:**
- Modify: `CLAUDE.md` ("Device trigger flow" section)
- Modify: `docs/runbooks/emqx-setup.md`

- [ ] **Step 1: Rewrite the CLAUDE.md flow section**

Replace steps 2-3 of "Device trigger flow (trigger-only model)" with the MQTT contract:

```markdown
2. **Trigger**: an authenticated caller (API key with the `devices:trigger`
   scope, plus a required `Idempotency-Key` header) does
   `POST /api/v1/devices/{deviceId}/trigger` with body
   `{ action: "show_qr", payload: { url } }`. The route checks device
   ownership/online status, reserves 1 credit (`lib/credits.ts` `reserveCredit`),
   enqueues a `deviceCommand` row, and publishes it to the device's MQTT topic.
   A failed publish fails the request closed (503, reservation cancelled) —
   MQTT is the only transport and there is no fallback.
3. **Deliver + render + ack**: the device is subscribed to `d/{deviceId}/cmd`,
   renders a QR from `payload.url`, and publishes an ack on `d/{deviceId}/ack`.
   EMQX forwards it to `POST /api/mqtt/ack`, which settles the reserved credit
   (`settleHold`); a failure or expiry releases it (`releaseHold`).

**Device transport, in full.** MQTT carries commands, acks, heartbeat, presence,
config and the OTA manifest. Config and the manifest ride the `cmd` topic as
payload-carrying `config-changed` / `firmware-update` commands, generated at
publish time because they embed short-lived presigned R2 URLs — never stored on
the command row. The device asks for config once per connect by publishing to
`d/{deviceId}/cfg/get`; it never polls. HTTPS survives only for the one-time
`GET /api/device/claim` bootstrap, R2 asset fetches and the OTA binary download.
See `docs/superpowers/specs/2026-07-29-mqtt-only-device-transport-design.md`.
```

- [ ] **Step 2: Note the health card in the runbook**

Add a line to the runbook's verification section pointing at the admin MQTT channel card as the way to confirm all four rules are live.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md docs/runbooks/emqx-setup.md
git commit -m "docs: describe the MQTT-only device transport"
```

---

### Task C5: Deploy Phase C and verify

- [ ] **Step 1: Full gate**

Run: `npm test && npx tsc --noEmit && npm run build`
Expected: all PASS

- [ ] **Step 2: Deploy**

Run: `vercel --prod --yes`

- [ ] **Step 3: Verify the deleted routes are gone**

Run: `curl -s -o /dev/null -w "%{http_code}\n" https://ditto-admin-brown.vercel.app/api/device/commands`
Expected: 404

- [ ] **Step 4: Verify the device is unaffected**

The device must still be online, still show the right branding, and still accept a trigger. Change the brand colour once more and confirm it repaints.

- [ ] **Step 5: Verify the trigger hard-fail is reachable but not firing**

Confirm real triggers still return 202 and settle. Do not deliberately break EMQX in production to test the 503 — that path was exercised during Phase B6 step 7.

- [ ] **Step 6: Report**

Report: deployment URL, the 404 on the deleted route, device online + firmware version, all four MQTT channels live, and one successful trigger with its credit settled.

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| One new topic `d/{id}/cfg/get` | B3 (device), A9 (rule), A4 (webhook) |
| Config + manifest on the existing `cmd` topic as command types | A2, B4 |
| Boot/reconnect config request | B3 |
| Config change publishes the config itself | A5 |
| Firmware published → push manifest to every claimed device | A7 |
| hb version reconcile catches a missed OTA | A6 |
| Nudge removed | A5 |
| Presign freshness / no retained messages | A2 (built per publish), A6 (rebuild on republish) |
| Delete 4 routes + `lib/device-auth.ts` | C1 |
| Claim stays HTTPS | C1 step 1 (explicitly asserted) |
| Firmware deletions incl. dead `cloud_post_document` | B5 |
| `cloud_config_parse_body` extraction | B1 |
| `ota_start_with_manifest` | B4 |
| Trigger hard-fail on publish failure | C2 |
| `config`/`ota` are desired-state, may stay pending | A4, A6, A7 (rows inserted pending; only trigger fails closed) |
| Online status from hb + presence, `OFFLINE_MINUTES` unchanged | no task needed — verified unchanged at `lib/device-status.ts:4` |
| hb republish stays load-bearing | A6 |
| No broker-outage screen | B5 step 3 (offline branch just waits) |
| R2 mitigation: `mqttWebhookPing` + admin card | A1, A8 |
| R3 phase ordering | phase structure + B7 step 3 gate |
| R4 retire legacy rows | C3 |
| R5 boot-storm jitter | B3 step 3 |
| Tests: webhook secret/identity/spoof, config-push freshness, ota boundaries, trigger failure, firmware parse | A1, A2, A3, B1, B2 |
| Docs: runbook + CLAUDE.md | A9, C4 |

Two spec test items are covered differently than written and this is deliberate: this repo has **no route integration tests** (all 45 test files are pure-function tests in `lib/`), so webhook secret/identity/spoof behaviour is tested at the parser level (A3) and the route keeps that logic to three lines copied from the proven heartbeat route. The trigger publish-failure path likewise has no unit test — it is verified on hardware in B6 step 7 (broker ban drill). Adding this repo's first route-test harness is real work that does not belong inside a transport migration.

**Placeholder scan:** No TBD/TODO. Three tasks legitimately begin with a read step because the target code was not read during planning — A7 step 1 (manual-command server action), A8 step 1 (host admin page), B4 step 1 (`ota.c` internals) and B1 step 1 (cfg-harness layout). Each names exactly what to find and the grep or command to find it.

**Type consistency:** `PushTarget` (A2) is satisfied by the selects in A4, A5 and A6 — all five fields present in each. `publishConfigCommand(dev, commandId)` takes the target object; `publishOtaCommand(deviceId, commandId)` takes the id, since it needs no pin context. `republishKindFor` returns exactly the three literals the A6 switch handles. `MqttChannel` values in A1 match the `recordWebhookPing` call sites in A1 step 7 and A4. On the device, `mqtt_parse_pending_config(device_config_t *)` / `mqtt_parse_pending_ota(fw_manifest_t *)` mirror each other and are consumed in B4 step 6.
