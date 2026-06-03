# Phase 2 — Device Remote Control + Offline Detection Design

_Last updated: 2026-06-03_

## Context

Devices (kiosks) only ever talk **to** the server: `POST /api/ingest` with
`Authorization: Bearer <deviceKey>` (SHA-256 hashed-key lookup). There is **no
server→device channel**, the app runs **serverless** (no persistent
connections), and nothing currently marks a device offline (`device.status` is
set to `online` on ingest and only flipped to `paused` by an admin).

This feature adds (1) **offline detection** — truthful online/offline status
from `lastSeenAt` freshness — and (2) **remote control** — a server-side
**command queue the device polls over HTTP** (reboot/refresh/identify), plus
app-version reporting.

> **Why polling, not MQTT:** MQTT needs an always-on broker + a firmware MQTT
> client and doesn't fit Vercel serverless. A command queue the device pulls on
> a timer reuses the existing HTTP + device-key model with zero new infra;
> minute-scale latency is fine for kiosk commands. The poll endpoint doubles as
> a heartbeat.

## Decisions (locked during brainstorming)

| Decision | Choice |
|---|---|
| Scope | Offline detection **+** command queue (both) |
| Command channel | **HTTP command queue, device polls** (approach A) — not MQTT/SSE |
| Offline detection | **Effective status computed from `lastSeenAt` at read time** (no cron); persisting via a sweep deferred until Vercel cron exists |
| Offline threshold | **15 minutes** (same as the health "stale" line) |
| Command authority | **platform_admin → any device; tenant owner/admin → own org's devices** |
| Protocol | `reboot`, `refresh`, `identify` commands **+** app-version reporting |
| Refactor | Extract `/api/ingest`'s bearer-key auth into shared `authenticateDevice` |

## Goals

1. Device online/offline status is truthful everywhere it's shown (derived from
   `lastSeenAt`), and "last seen" is visible in the fleet UI.
2. An authorized user can enqueue `reboot`/`refresh`/`identify` for a device; the
   device picks it up by polling and acknowledges.
3. Devices report their app version, shown in the fleet.
4. The device-facing protocol is documented for firmware authors.
5. Pure status/format/validation logic is unit-tested.

## Non-Goals

- Real-time push (MQTT/SSE) — explicitly rejected.
- Persisting offline status via a background sweep (deferred to when Vercel cron exists).
- The kiosk **firmware** changes — out of this repo. Remote control is **inert
  until firmware polls `/api/device/commands`**; we ship the server contract + doc.

---

## Architecture

### 1. Offline detection (pure + read-time application)

`lib/device-status.ts` (pure, IO-free):

```ts
export const OFFLINE_MINUTES = 15;
export type DeviceStatus = "online" | "offline" | "paused";

/** Truthful status: paused wins; else offline if not seen recently; else stored. */
export function effectiveDeviceStatus(
  storedStatus: string,
  lastSeenAt: Date | null,
  now: Date,
  offlineMinutes = OFFLINE_MINUTES,
): DeviceStatus {
  if (storedStatus === "paused") return "paused";
  if (!lastSeenAt) return "offline";
  return now.getTime() - lastSeenAt.getTime() > offlineMinutes * 60_000
    ? "offline"
    : "online";
}

/** Human "last seen" string. */
export function formatLastSeen(lastSeenAt: Date | null, now: Date): string {
  if (!lastSeenAt) return "never";
  const mins = Math.floor((now.getTime() - lastSeenAt.getTime()) / 60_000);
  if (mins < 1) return "just now";
  if (mins < 60) return `${mins}m ago`;
  const hrs = Math.floor(mins / 60);
  if (hrs < 24) return `${hrs}h ago`;
  return `${Math.floor(hrs / 24)}d ago`;
}
```

**Applied at read time** in the device-facing data fns — `getAllDevices`
(admin fleet), the tenant store/device device lists, `getDevice` (detail), and
the **health dashboard fleet counts** (`getPlatformHealth` derives
`online/offline/paused` from effective status instead of stored status). Each
maps `status` → `effectiveDeviceStatus(status, lastSeenAt, now)` and includes a
`lastSeen` display string. Stored `device.status` is unchanged.

### 2. Command queue

`lib/db/schema.ts` — new table:

```ts
export const deviceCommand = pgTable("device_command", {
  id: text("id").primaryKey(),
  deviceId: text("device_id").notNull().references(() => device.id, { onDelete: "cascade" }),
  organizationId: text("organization_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
  type: text("type", { enum: ["reboot", "refresh", "identify"] }).notNull(),
  status: text("status", { enum: ["pending", "delivered", "acked", "failed"] }).default("pending").notNull(),
  result: text("result"),
  createdByUserId: text("created_by_user_id"),
  createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  deliveredAt: timestamp("delivered_at"),
  ackedAt: timestamp("acked_at"),
}, (t) => [index("device_command_device_status_idx").on(t.deviceId, t.status)]);
```

Plus `device.appVersion text` (nullable) column.

Pure helpers `lib/device-commands.ts`: `COMMAND_TYPES`, `isValidCommandType(t)`.

**Shared device auth** `lib/device-auth.ts`:

```ts
export async function authenticateDevice(req: Request): Promise<Device | null>
```

Extracted from `/api/ingest`: parse `Authorization: Bearer`, hash, look up by
`deviceKeyHash`. `/api/ingest` is refactored to call it; the command endpoints reuse it.

**Endpoints** (device-key auth, `runtime = "nodejs"`):
- `GET /api/device/commands` — authenticate; bump `lastSeenAt = now`, status =
  `online`; if `X-Device-Version` header present, store `device.appVersion`;
  return `pending` commands for the device and mark them `delivered`
  (`deliveredAt = now`). Paused devices: still return commands (so a paused
  device can be rebooted/unpaused), but never auto-set online if paused — keep
  paused. (Bump `lastSeenAt` regardless.)
- `POST /api/device/commands/ack` — body `{ commandId, ok, result? }`; scope to a
  command belonging to the authenticated device; set `status = ok ? "acked" : "failed"`,
  `ackedAt = now`, `result`.

**Enqueue** server action `lib/actions/device-commands.ts` →
`enqueueDeviceCommand(deviceId, type)`:
- `isValidCommandType(type)` else error.
- Authorize: `getContext()`; if `user.role === "platform_admin"` → any device;
  else the device's org must be one where the user is `owner`/`admin`
  (`ctx.organizations`). Load the device to get its `organizationId`.
- Insert `pending` command with `createdByUserId`; `recordAudit`
  (`device.command_enqueued`, metadata `{ type }`); `revalidatePath` the device views.

### 3. UI

- Device detail page (`/tenant/stores/[storeId]/[deviceId]` and the admin device
  views) gains a **command bar**: Reboot / Refresh / Identify buttons (shown only
  if the viewer can command the device) → confirm → `enqueueDeviceCommand`.
- A **recent commands** list (type, status, created, acked) via
  `getDeviceCommands(deviceId)`.
- Fleet/detail show effective status + "last seen" + `appVersion`.

### 4. Protocol doc

`docs/device-protocol.md` — for firmware authors: the three device endpoints
(`POST /api/ingest`, `GET /api/device/commands`, `POST /api/device/commands/ack`),
bearer-key auth, the `X-Device-Version` header, command types, recommended poll
interval, and the ack contract.

### 5. Audit

`AUDIT.deviceCommandEnqueued = "device.command_enqueued"` recorded on enqueue.

---

## Data model

One additive migration: `device_command` table + `device.appVersion` column +
the command index. No enum changes to `device.status`.

## Error handling

- Unknown/invalid device key on the device endpoints → 401 (reuse ingest's behavior).
- Invalid command type on enqueue → `{ ok:false, error }`.
- Ack for a command not belonging to the device → ignored (404/no-op).
- `effectiveDeviceStatus`/`formatLastSeen` are pure and total (handle null).

## Testing

- **Pure unit (TDD):** `effectiveDeviceStatus` (paused wins; null → offline;
  boundary at 15 min; fresh → online); `formatLastSeen` (never / just now / m / h / d);
  `isValidCommandType`.
- **Mocked/manual integration:** `GET /api/device/commands` returns + delivers
  pending commands and bumps lastSeenAt; ack flips status; enqueue authorization
  (platform admin any; tenant owner/admin own org only; member denied).
- **Manual:** enqueue a reboot from the UI → poll the endpoint with the device
  key (curl) → see it delivered → POST ack → status `acked`; age `lastSeenAt` →
  device shows offline in fleet + health.

## File structure

| File | Responsibility | New? |
|---|---|---|
| `lib/device-status.ts` | pure `effectiveDeviceStatus`, `formatLastSeen`, `OFFLINE_MINUTES` | Create |
| `lib/device-status.test.ts` | tests | Create |
| `lib/device-commands.ts` | pure `COMMAND_TYPES`, `isValidCommandType` | Create |
| `lib/device-commands.test.ts` | tests | Create |
| `lib/device-auth.ts` | shared `authenticateDevice(req)` | Create |
| `lib/db/schema.ts` | `device_command` table + `device.appVersion` | Modify |
| `app/api/ingest/route.ts` | use `authenticateDevice`; store `X-Device-Version` | Modify |
| `app/api/device/commands/route.ts` | poll (GET) — deliver + heartbeat | Create |
| `app/api/device/commands/ack/route.ts` | ack (POST) | Create |
| `lib/actions/device-commands.ts` | `enqueueDeviceCommand` server action | Create |
| `lib/data.ts` | `getDeviceCommands`; apply effective status in device fns + health | Modify |
| `lib/audit.ts` | + `deviceCommandEnqueued` | Modify |
| `components/devices/command-bar.tsx` | enqueue buttons (client) | Create |
| device detail page(s) | render command bar + history + last-seen/version | Modify |
| `docs/device-protocol.md` | firmware-facing protocol | Create |

## Sequencing

1. Pure helpers (`device-status`, `device-commands`) + tests.
2. Schema (`device_command` + `appVersion`) + migration.
3. `authenticateDevice` extraction; refactor `/api/ingest` to use it + version header.
4. Device endpoints: `GET /api/device/commands`, `POST .../ack`.
5. `enqueueDeviceCommand` action + audit + `getDeviceCommands`.
6. Apply effective status in device data fns + health fleet counts.
7. UI: command bar + history + last-seen/version on device detail; protocol doc.
8. Manual verification.
