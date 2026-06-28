# Phase 2A — Device Fleet Ops Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give platform admins per-device operational depth: an admin device detail page with remote commands, firmware visibility in the fleet table, and automated offline reconciliation folded into the daily health cron.

**Architecture:** Two pure helpers (`shouldMarkOffline`, `firmwareUpdateAvailable`) in `lib/device-status.ts`; an offline-reconcile step in the existing `evaluateAndPersistAlerts` (no new cron); a new admin device detail page reusing the existing `getDevice`/`getDeviceCommands`/`CommandBar`/`DeviceRowActions`; and a firmware column + detail link in `FleetTable`.

**Tech Stack:** Next.js 16 (RSC pages + client `FleetTable`), Drizzle/Neon, Vitest (pure helpers). Reuses the existing `deviceCommand` queue, `firmwareRelease` table, and `enqueueDeviceCommand` action.

## Global Constraints

- **Offline reconcile is folded into the EXISTING daily health cron** (`/api/cron/health` → `evaluateAndPersistAlerts`). No new Vercel cron.
- **Reconcile flips `online → offline` only**, never `paused`; uses the same `OFFLINE_MINUTES = 15` threshold as `effectiveDeviceStatus`; audits each flip with a new `AUDIT.deviceWentOffline = "device.went_offline"`; idempotent.
- **Offline *email* is OUT of scope (→ 2B).** The existing `computeAlerts` stale-device alert row already surfaces it; do not add a separate offline-email path.
- **Command-queue pause is OUT of scope.** Keep the existing direct-write pause (`DeviceRowActions` / `setDeviceActiveAdmin`).
- **All admin device surfaces use `effectiveDeviceStatus`** for read-time accuracy. `getAllDevices` already applies it; the new admin detail page computes it for its status badge.
- Verification per task: `npm run test` (currently 278, stays green), `npm run build`, `npx tsc --noEmit`. Dev server on **:3001**. UI checks are deferred/manual.

## File Map

| File | Responsibility | Task |
|---|---|---|
| `lib/device-status.ts` (modify) + `.test.ts` | pure `shouldMarkOffline` + `firmwareUpdateAvailable` | 1 |
| `lib/audit.ts` (modify) | `AUDIT.deviceWentOffline` | 2 |
| `lib/alerts-sync.ts` (modify) | `reconcileOfflineDevices` + call it in `evaluateAndPersistAlerts` | 2 |
| `app/(admin)/admin/devices/[deviceId]/page.tsx` (new) | admin device detail page | 3 |
| `components/fleet-table.tsx` (modify) | Firmware column + update badge + Device-ID link | 4 |
| `app/(admin)/admin/devices/page.tsx` (modify) | pass `latestFirmwareVersion` to `FleetTable` | 4 |

---

### Task 1: Pure fleet helpers

**Files:**
- Modify: `lib/device-status.ts`
- Test: `lib/device-status.test.ts`

**Interfaces:**
- Consumes: existing `OFFLINE_MINUTES = 15` (same file).
- Produces:
  - `shouldMarkOffline(d: { status: string; lastSeenAt: Date | null }, now: Date, offlineMinutes?: number): boolean`
  - `firmwareUpdateAvailable(deviceVersion: string | null, latestVersion: string | null): boolean`

- [ ] **Step 1: Write the failing test**

Create `lib/device-status.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import {
  effectiveDeviceStatus,
  shouldMarkOffline,
  firmwareUpdateAvailable,
  OFFLINE_MINUTES,
} from "./device-status";

const now = new Date("2026-06-28T12:00:00Z");
const stale = new Date(now.getTime() - (OFFLINE_MINUTES + 1) * 60_000);
const fresh = new Date(now.getTime() - 60_000);

describe("effectiveDeviceStatus (existing, sanity)", () => {
  it("paused wins", () => {
    expect(effectiveDeviceStatus("paused", stale, now)).toBe("paused");
  });
});

describe("shouldMarkOffline", () => {
  it("flips an online device that is stale", () => {
    expect(shouldMarkOffline({ status: "online", lastSeenAt: stale }, now)).toBe(true);
  });
  it("does NOT flip a fresh online device", () => {
    expect(shouldMarkOffline({ status: "online", lastSeenAt: fresh }, now)).toBe(false);
  });
  it("does NOT flip a paused device even if stale", () => {
    expect(shouldMarkOffline({ status: "paused", lastSeenAt: stale }, now)).toBe(false);
  });
  it("does NOT flip an already-offline device", () => {
    expect(shouldMarkOffline({ status: "offline", lastSeenAt: stale }, now)).toBe(false);
  });
  it("flips an online device that was never seen (null lastSeenAt)", () => {
    expect(shouldMarkOffline({ status: "online", lastSeenAt: null }, now)).toBe(true);
  });
});

describe("firmwareUpdateAvailable", () => {
  it("true when latest differs from device version", () => {
    expect(firmwareUpdateAvailable("2.4.1", "2.5.0")).toBe(true);
  });
  it("false when equal", () => {
    expect(firmwareUpdateAvailable("2.5.0", "2.5.0")).toBe(false);
  });
  it("false when there is no latest release", () => {
    expect(firmwareUpdateAvailable("2.4.1", null)).toBe(false);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run lib/device-status.test.ts`
Expected: FAIL — `shouldMarkOffline`/`firmwareUpdateAvailable` are not exported.

- [ ] **Step 3: Add the helpers**

Append to `lib/device-status.ts` (keep the existing `OFFLINE_MINUTES`, `DeviceStatus`, `effectiveDeviceStatus`):

```ts
/** Should this device's STORED status be reconciled to "offline"? True only for
 * an "online" row whose lastSeenAt is older than the threshold (or never seen).
 * Never flips "paused" or an already-"offline" row. Mirrors effectiveDeviceStatus
 * but operates on the raw stored status for the daily reconcile sweep. */
export function shouldMarkOffline(
  d: { status: string; lastSeenAt: Date | null },
  now: Date,
  offlineMinutes = OFFLINE_MINUTES,
): boolean {
  if (d.status !== "online") return false;
  if (!d.lastSeenAt) return true;
  return now.getTime() - d.lastSeenAt.getTime() > offlineMinutes * 60_000;
}

/** Is a newer firmware available? False when there is no latest release. */
export function firmwareUpdateAvailable(
  deviceVersion: string | null,
  latestVersion: string | null,
): boolean {
  return latestVersion != null && deviceVersion !== latestVersion;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run lib/device-status.test.ts`
Expected: PASS (all cases).

- [ ] **Step 5: Full suite + typecheck**

Run: `npm run test && npx tsc --noEmit`
Expected: all green (278 + new), no type errors.

- [ ] **Step 6: Commit**

```bash
git add lib/device-status.ts lib/device-status.test.ts
git commit -m "feat(devices): pure shouldMarkOffline + firmwareUpdateAvailable helpers (2A)"
```

---

### Task 2: Offline reconciliation in the health cron

**Files:**
- Modify: `lib/audit.ts`
- Modify: `lib/alerts-sync.ts`

**Interfaces:**
- Consumes: `shouldMarkOffline` (Task 1); `device` table; `recordAudit`/`AUDIT` (`lib/audit.ts`).
- Produces: `reconcileOfflineDevices(now: Date): Promise<number>` (count flipped); it runs as the first step of `evaluateAndPersistAlerts`.

- [ ] **Step 1: Add the audit constant**

In `lib/audit.ts`, inside the `AUDIT` object after `deviceClaimed: "device.claimed",` add:

```ts
  deviceWentOffline: "device.went_offline",
```

- [ ] **Step 2: Implement `reconcileOfflineDevices` and call it**

In `lib/alerts-sync.ts`, add imports (extend the existing `./db/schema` import to include `device`, and add the helper + audit imports):

```ts
import { alert as alertTable, user as userTable, device as deviceTable } from "./db/schema";
import { shouldMarkOffline } from "./device-status";
import { recordAudit, AUDIT } from "./audit";
```

Add the function (above `evaluateAndPersistAlerts`):

```ts
/** Reconcile stored device status: flip "online" rows that have gone stale to
 * "offline" (never touches "paused"/"offline"), and audit each flip. Idempotent.
 * Folded into the daily health sweep so no separate cron is needed. */
export async function reconcileOfflineDevices(now: Date): Promise<number> {
  const onlineRows = await db
    .select({
      id: deviceTable.id,
      organizationId: deviceTable.organizationId,
      status: deviceTable.status,
      lastSeenAt: deviceTable.lastSeenAt,
    })
    .from(deviceTable)
    .where(eq(deviceTable.status, "online"));

  const toFlip = onlineRows.filter((r) => shouldMarkOffline(r, now));
  if (toFlip.length === 0) return 0;

  await db
    .update(deviceTable)
    .set({ status: "offline" })
    .where(inArray(deviceTable.id, toFlip.map((r) => r.id)));

  for (const r of toFlip) {
    await recordAudit({
      organizationId: r.organizationId,
      actor: { type: "system" },
      action: AUDIT.deviceWentOffline,
      target: { type: "device", id: r.id },
      metadata: { lastSeenAt: r.lastSeenAt ? r.lastSeenAt.toISOString() : null },
    });
  }
  return toFlip.length;
}
```

(`eq` and `inArray` are already imported at the top of this file.)

- [ ] **Step 3: Run it at the start of `evaluateAndPersistAlerts`**

In `lib/alerts-sync.ts`, the function currently begins:

```ts
export async function evaluateAndPersistAlerts(): Promise<{
  opened: number;
  resolved: number;
  stillOpen: number;
}> {
  const current = computeAlerts(await getAlertInputs());
```

Insert a reconcile call as the very first statement so the daily sweep reconciles status before computing alerts (the stale-device alert then reflects the just-reconciled fleet):

```ts
export async function evaluateAndPersistAlerts(): Promise<{
  opened: number;
  resolved: number;
  stillOpen: number;
}> {
  await reconcileOfflineDevices(new Date());
  const current = computeAlerts(await getAlertInputs());
```

- [ ] **Step 4: Typecheck + build + suite**

Run: `npx tsc --noEmit && npm run build && npm run test`
Expected: no type errors; build OK; 278+ green (no existing test drives this IO path).

- [ ] **Step 5: Commit**

```bash
git add lib/audit.ts lib/alerts-sync.ts
git commit -m "feat(devices): reconcile online->offline in the daily health sweep + audit (2A)"
```

---

### Task 3: Admin device detail page

**Files:**
- Create: `app/(admin)/admin/devices/[deviceId]/page.tsx`

**Interfaces:**
- Consumes: `getDevice(deviceId): Promise<{ device: Device; store: Store; tenant: Tenant } | null>` and `getDeviceCommands(deviceId)` (`lib/data.ts`); `firmwareRelease` table; `effectiveDeviceStatus` + `firmwareUpdateAvailable` (Task 1); `CommandBar` (`components/devices/command-bar.tsx`); `DeviceRowActions` (`components/device-row-actions.tsx`); `requirePlatformAdmin` (`lib/session.ts`); `StatusDot` (`components/status-badge`).

- [ ] **Step 1: Create the page**

Create `app/(admin)/admin/devices/[deviceId]/page.tsx`:

```tsx
import Link from "next/link";
import { notFound } from "next/navigation";
import { ArrowLeft, Cable, Cpu, Globe, HardDrive, FileText, Wifi } from "lucide-react";
import { desc } from "drizzle-orm";
import { PageHeader } from "@/components/page-header";
import { KpiCard } from "@/components/kpi-card";
import { StatusDot } from "@/components/status-badge";
import { DeviceRowActions } from "@/components/device-row-actions";
import { CommandBar } from "@/components/devices/command-bar";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { getDevice, getDeviceCommands } from "@/lib/data";
import { db } from "@/lib/db";
import { firmwareRelease } from "@/lib/db/schema";
import { requirePlatformAdmin } from "@/lib/session";
import { effectiveDeviceStatus, firmwareUpdateAvailable, type DeviceStatus } from "@/lib/device-status";
import { formatNumber, timeAgo } from "@/lib/format";

export default async function AdminDeviceDetailPage({
  params,
}: {
  params: Promise<{ deviceId: string }>;
}) {
  await requirePlatformAdmin();
  const { deviceId } = await params;
  const result = await getDevice(deviceId);
  if (!result) notFound();

  const { device, store, tenant } = result;
  const commands = await getDeviceCommands(device.id);

  const [latestFw] = await db
    .select({ version: firmwareRelease.version })
    .from(firmwareRelease)
    .orderBy(desc(firmwareRelease.createdAt))
    .limit(1);
  const updateAvailable = firmwareUpdateAvailable(device.firmwareVersion, latestFw?.version ?? null);

  const status: DeviceStatus = effectiveDeviceStatus(
    device.status,
    device.lastSeenAt ? new Date(device.lastSeenAt) : null,
    new Date(),
  );

  const specs: { icon: typeof Cpu; label: string; value: string; mono?: boolean }[] = [
    { icon: HardDrive, label: "Device ID", value: device.id, mono: true },
    { icon: Globe, label: "IP address", value: device.ipAddress, mono: true },
    {
      icon: device.connectionType === "wifi" ? Wifi : Cable,
      label: "Connection",
      value: device.connectionType === "wifi" ? "Wi-Fi" : "Ethernet",
    },
    {
      icon: Cpu,
      label: "Firmware",
      value: `v${device.firmwareVersion}${updateAvailable ? ` → v${latestFw!.version} available` : ""}`,
      mono: true,
    },
  ];

  return (
    <>
      <Link
        href="/admin/devices"
        className="inline-flex items-center gap-1.5 text-sm text-muted-foreground transition-colors hover:text-foreground"
      >
        <ArrowLeft className="size-4" />
        Device Fleet
      </Link>

      <PageHeader title={device.name} description={`Printer at ${store.name}`} />

      <div className="grid gap-6 lg:grid-cols-3">
        <div className="space-y-6 lg:col-span-2">
          <div className="grid gap-4 sm:grid-cols-2">
            <KpiCard label="Documents today" value={formatNumber(device.documentsToday)} icon={FileText} />
            <KpiCard label="Documents this month" value={formatNumber(device.documentsThisMonth)} icon={FileText} />
          </div>

          <Card>
            <CardHeader>
              <CardTitle className="text-base">Device details</CardTitle>
            </CardHeader>
            <CardContent className="grid gap-px overflow-hidden rounded-xl border sm:grid-cols-2">
              {specs.map((s) => (
                <div key={s.label} className="flex items-center gap-3 bg-card p-4">
                  <span className="flex size-9 items-center justify-center rounded-lg bg-accent text-accent-foreground">
                    <s.icon className="size-4" />
                  </span>
                  <div>
                    <p className="text-xs text-muted-foreground">{s.label}</p>
                    <p className={s.mono ? "font-mono text-sm" : "text-sm font-medium"}>{s.value}</p>
                  </div>
                </div>
              ))}
            </CardContent>
          </Card>
        </div>

        <div className="space-y-4">
          <Card>
            <CardHeader>
              <CardTitle className="text-base">Status & management</CardTitle>
            </CardHeader>
            <CardContent className="space-y-3 text-sm">
              <div className="flex items-center justify-between">
                <span className="text-muted-foreground">Status</span>
                <span className="inline-flex items-center gap-1.5 capitalize">
                  <StatusDot status={status} />
                  {status}
                </span>
              </div>
              <div className="flex justify-between">
                <span className="text-muted-foreground">Customer</span>
                <Link href={`/admin/customers/${tenant.id}`} className="font-medium underline">
                  {tenant.name}
                </Link>
              </div>
              <div className="flex justify-between">
                <span className="text-muted-foreground">Store</span>
                <span className="font-medium">{store.name}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-muted-foreground">Last seen</span>
                <span className="font-medium">{timeAgo(device.lastSeen)}</span>
              </div>
              <div className="flex items-center justify-between pt-1">
                <span className="text-muted-foreground">Actions</span>
                <DeviceRowActions deviceId={device.id} deviceName={device.name} status={status} />
              </div>
            </CardContent>
          </Card>
        </div>
      </div>

      <section className="flex flex-col gap-3">
        <h2 className="text-lg font-medium">Remote control</h2>
        <CommandBar deviceId={device.id} />
        {commands.length > 0 && (
          <table className="w-full text-sm">
            <thead>
              <tr className="text-left text-muted-foreground">
                <th className="py-2">Command</th>
                <th>Status</th>
                <th>Queued</th>
              </tr>
            </thead>
            <tbody>
              {commands.map((c) => (
                <tr key={c.id} className="border-t">
                  <td className="py-2">{c.type}</td>
                  <td>{c.status}</td>
                  <td>{c.createdAt.slice(0, 19).replace("T", " ")}</td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </section>
    </>
  );
}
```

- [ ] **Step 2: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors; build lists the new route `/admin/devices/[deviceId]`. (If `DeviceRowActions`'s `status` prop is typed narrower than `DeviceStatus`, pass `status` as-is — it is one of online/offline/paused; adjust only if tsc complains.)

- [ ] **Step 3: Commit**

```bash
git add "app/(admin)/admin/devices/[deviceId]/page.tsx"
git commit -m "feat(devices): admin device detail page with CommandBar + firmware + history (2A)"
```

---

### Task 4: Fleet table firmware column + detail link

**Files:**
- Modify: `components/fleet-table.tsx`
- Modify: `app/(admin)/admin/devices/page.tsx`

**Interfaces:**
- Consumes: `firmwareUpdateAvailable` (Task 1); `DeviceRow` (has `firmwareVersion: string`); `firmwareRelease` table.
- Produces: `FleetTable` gains a `latestFirmwareVersion: string | null` prop.

- [ ] **Step 1: Pass the latest firmware version from the admin devices page**

In `app/(admin)/admin/devices/page.tsx`, add a read of the latest firmware release and pass it to `FleetTable`. Update the imports + body:

```tsx
import { desc } from "drizzle-orm";
import { Cpu } from "lucide-react";
import { PageHeader } from "@/components/page-header";
import { KpiCard } from "@/components/kpi-card";
import { FleetTable } from "@/components/fleet-table";
import { getAllDevices, getTenants } from "@/lib/data";
import { db } from "@/lib/db";
import { firmwareRelease } from "@/lib/db/schema";

export default async function FleetPage() {
  const rows = await getAllDevices();
  const customers = (await getTenants()).map((t) => ({ id: t.id, name: t.name }));
  const [latestFw] = await db
    .select({ version: firmwareRelease.version })
    .from(firmwareRelease)
    .orderBy(desc(firmwareRelease.createdAt))
    .limit(1);
  const online = rows.filter((r) => r.status === "online").length;
  const paused = rows.filter((r) => r.status === "paused").length;
  const offline = rows.filter((r) => r.status === "offline").length;

  return (
    <>
      <PageHeader
        title="Device Fleet"
        description="Every printer across every customer, in one place."
      />

      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <KpiCard label="Total devices" value={String(rows.length)} icon={Cpu} />
        <KpiCard label="Online" value={String(online)} hint="accepting documents" />
        <KpiCard label="Paused" value={String(paused)} hint="temporarily off" />
        <KpiCard label="Offline" value={String(offline)} hint="unreachable" />
      </div>

      <FleetTable rows={rows} customers={customers} latestFirmwareVersion={latestFw?.version ?? null} />
    </>
  );
}
```

- [ ] **Step 2: Add the Firmware column + Device-ID link to `FleetTable`**

In `components/fleet-table.tsx`:

a) Add imports near the top (after the existing imports):

```tsx
import Link from "next/link";
import { firmwareUpdateAvailable } from "@/lib/device-status";
```

b) Add the prop to the component signature:

```tsx
export function FleetTable({
  rows,
  customers,
  latestFirmwareVersion,
}: {
  rows: DeviceRow[];
  customers: { id: string; name: string }[];
  latestFirmwareVersion: string | null;
}) {
```

c) Add a `Firmware` header after the `Last seen` header:

```tsx
              <TableHead>Last seen</TableHead>
              <TableHead>Firmware</TableHead>
              <TableHead className="text-right">Documents (mo.)</TableHead>
```

d) Make the Device ID cell a link, and add the firmware cell. Replace the existing Device ID `<TableCell>` and add a firmware `<TableCell>` after the Last-seen cell:

```tsx
                <TableCell className="pl-6 font-mono text-xs">
                  <Link href={`/admin/devices/${r.id}`} className="underline-offset-2 hover:underline">
                    {r.id}
                  </Link>
                </TableCell>
```

```tsx
                <TableCell className="text-muted-foreground">
                  {timeAgo(r.lastSeen)}
                </TableCell>
                <TableCell className="font-mono text-xs">
                  v{r.firmwareVersion}
                  {firmwareUpdateAvailable(r.firmwareVersion, latestFirmwareVersion) && (
                    <span className="ml-1.5 rounded bg-amber-100 px-1.5 py-0.5 text-[10px] font-medium text-amber-800 dark:bg-amber-950/50 dark:text-amber-300">
                      update
                    </span>
                  )}
                </TableCell>
```

e) The empty-state row spans the table; bump its `colSpan` from `7` to `8` (one new column):

```tsx
                <TableCell
                  colSpan={8}
                  className="py-12 text-center text-sm text-muted-foreground"
                >
                  No devices match your filters.
                </TableCell>
```

- [ ] **Step 3: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: no type errors; build OK. (`FleetTable` is the only caller of itself — the admin page now passes `latestFirmwareVersion`, so no other call site needs updating; confirm grep `FleetTable` shows just the admin page + the component.)

- [ ] **Step 4: Commit**

```bash
git add components/fleet-table.tsx "app/(admin)/admin/devices/page.tsx"
git commit -m "feat(devices): firmware column + update badge + detail link in the fleet table (2A)"
```

---

## Deferred acceptance (manual UI / cron — controller/user)

- `/admin/devices` shows the Firmware column + "update" badge; the Device-ID links to the detail page.
- `/admin/devices/[id]` renders status/specs/firmware, the Status & management card (with `DeviceRowActions`), the `CommandBar`, and command history; issuing a command enqueues a `deviceCommand` row.
- Trigger `/api/cron/health` (with `CRON_SECRET`) and confirm a long-stale `online` device flips to `offline` + an audit `device.went_offline` row appears.

---

## Self-Review

**Spec coverage:**
- `shouldMarkOffline` + `firmwareUpdateAvailable` pure helpers (spec §A) → Task 1. ✅
- Offline reconcile folded into health cron + `AUDIT.deviceWentOffline`, never touches paused, idempotent (spec §B + decisions 1) → Task 2. ✅
- Admin device detail page with CommandBar/pause/firmware/history (spec §C) → Task 3. ✅
- Fleet firmware column + update badge + detail link (spec §D) → Task 4. ✅
- Offline email out of scope (no email added); command-queue pause out of scope (reuses DeviceRowActions) — honored across Tasks 2–3. ✅
- Testing: pure unit tests (Task 1) + deferred manual UI/cron checks (spec §Testing). ✅

**Placeholder scan:** None — every code step shows complete code. ✅

**Type consistency:** `shouldMarkOffline`/`firmwareUpdateAvailable` signatures (Task 1) match their uses in Tasks 2/3/4; `effectiveDeviceStatus` returns `DeviceStatus` used for the badge + `DeviceRowActions` prop; `latestFirmwareVersion: string | null` prop added in Task 4 and supplied by the admin page; `getDevice` returns `{device, store, tenant}` consumed by Task 3; `DeviceRow.firmwareVersion` used in Task 4. ✅

**Note for implementers:** `reconcileOfflineDevices` deliberately loads `status='online'` rows and filters them in JS with `shouldMarkOffline` (so the unit-tested rule is the single source of truth and is load-bearing in production), rather than duplicating the threshold in raw SQL. The online set is small relative to the fleet; this is intentional, not an oversight.
