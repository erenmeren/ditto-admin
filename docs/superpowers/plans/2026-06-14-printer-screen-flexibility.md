# Printer Screen Flexibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-clock display options (hide date, hide weekday, alignment), a one-click "Insert top bar" preset, and remove the "Lane 1"/tagline placeholders from the idle default — all within the existing v3 `PrinterConfig` object model.

**Architecture:** Pure-function changes to `lib/printer-layout.ts` (seed, a `clock?` sub-config on `PrinterObject` mirroring the existing `icon?` pattern, normalize defaults, a `topBarArrangement()` helper) drive small renderer/editor/UI changes. No DB migration — everything rides in the `tenant_settings.kiosk_screens` jsonb, with `normalizePrinterConfig` supplying defaults so existing configs are unchanged and keep today's look.

**Tech Stack:** Next.js 16, React 19, TypeScript strict, Tailwind v4, vitest 3 (pure-function tests; UI tasks verify via `npx tsc --noEmit` + `npm run build` + dev-server smoke). Spec: `docs/superpowers/specs/2026-06-14-printer-screen-flexibility-design.md`.

---

## File Structure

| File | Change |
|---|---|
| `lib/printer-layout.ts` | `seededText()` removed (idle default loses the two labels); `PrinterClockOptions` type + `clock?` on `PrinterObject`; `sanitizeClock()` + clock branch in `sanitizeObject` (align + clock); `topBarArrangement()` helper. |
| `lib/printer-layout.test.ts` | idle-seed assertion (0 text); normalize clock-defaults + bad input; `topBarArrangement` shape. |
| `components/device-preview/printer-clock.tsx` | `showDate`/`showWeekday`/`align` props + conditional date formatting. |
| `components/device-preview/printer-preview.tsx` | `ClockObject` passes clock options + align through and honors align in its flex wrapper. |
| `components/device-preview/printer-editor/use-printer-editor.ts` | `insertTopBar()` action + on the `PrinterEditor` interface. |
| `components/device-preview/printer-editor/printer-controls.tsx` | clock Properties: Align + Show date + Show weekday; "Insert top bar" button. |

No change needed in `branding-editor.tsx` (it already passes `editor` to `PrinterControls`; `insertTopBar` is on the editor). No change in `data.ts`/`actions.ts`/schema (jsonb config; normalize handles the new field).

---

### Task 1: Remove the "Lane 1" / tagline placeholders from the idle default

**Files:**
- Modify: `lib/printer-layout.ts`
- Test: `lib/printer-layout.test.ts`

- [ ] **Step 1: Update the failing test**

In `lib/printer-layout.test.ts`, find the `seededScreen` idle test (it currently asserts two text objects, e.g. `expect(idle.filter((o) => o.type === "text").length).toBe(2)`). Change it to assert **no** text objects, widgets intact:

```ts
  it("seeds idle with logo/clock/wifi and no placeholder text labels", () => {
    const idle = seededScreen("idle").objects;
    expect(idle.some((o) => o.type === "logo")).toBe(true);
    expect(idle.some((o) => o.type === "clock")).toBe(true);
    expect(idle.some((o) => o.type === "wifi")).toBe(true);
    expect(idle.filter((o) => o.type === "text").length).toBe(0);
  });
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run printer-layout`
Expected: FAIL — idle currently has 2 text objects.

- [ ] **Step 3: Remove the seeded labels**

In `lib/printer-layout.ts`, delete the `seededText()` function (the block returning `text-lane` + `text-tagline`), and change `defaultLayout()`'s objects from `[...fixed, ...seededText()]` to:

```ts
    objects: [...fixed],
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run printer-layout`
Expected: PASS. Then `npx tsc --noEmit` — no errors (confirm nothing else referenced `seededText`).

- [ ] **Step 5: Commit**

```bash
git add lib/printer-layout.ts lib/printer-layout.test.ts
git commit -m "feat(branding): drop Lane 1 / tagline from the idle default seed"
```

---

### Task 2: Per-clock display options — type + normalize

**Files:**
- Modify: `lib/printer-layout.ts`
- Test: `lib/printer-layout.test.ts`

- [ ] **Step 1: Write the failing test (extend `printer-layout.test.ts`)**

```ts
import { normalizePrinterConfig } from "./printer-layout";

describe("clock options", () => {
  const cfg = (clockObj: Record<string, unknown>) => ({
    version: 3, clockTimezone: "UTC", clock24h: false, wifiLevel: 3,
    screens: { idle: { objects: [clockObj] } },
  });
  const idleClock = (raw: unknown) =>
    normalizePrinterConfig(raw).screens.idle.objects.find((o) => o.type === "clock")!;

  it("defaults clock options to shown + center align", () => {
    const c = idleClock(cfg({ id: "clock", type: "clock", x: 0.25, y: 0.5, w: 0.5, h: 0.18, visible: true, z: 0 }));
    expect(c.clock).toEqual({ showDate: true, showWeekday: true });
    expect(c.align).toBe("center");
  });

  it("preserves explicit clock options + align", () => {
    const c = idleClock(cfg({ id: "clock", type: "clock", x: 0.25, y: 0.5, w: 0.5, h: 0.18, visible: true, z: 0, align: "left", clock: { showDate: false, showWeekday: false } }));
    expect(c.clock).toEqual({ showDate: false, showWeekday: false });
    expect(c.align).toBe("left");
  });

  it("coerces a garbage clock field to defaults", () => {
    const c = idleClock(cfg({ id: "clock", type: "clock", x: 0.25, y: 0.5, w: 0.5, h: 0.18, visible: true, z: 0, clock: "nope", align: 99 }));
    expect(c.clock).toEqual({ showDate: true, showWeekday: true });
    expect(c.align).toBe("center");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run printer-layout`
Expected: FAIL — the clock object comes back with no `clock`/`align` (current widget branch only sets the box).

- [ ] **Step 3: Implement the type + normalize branch**

In `lib/printer-layout.ts`:

(a) After the `PrinterIcon` interface (just before `export type TextAlign`), add:

```ts
export interface PrinterClockOptions {
  showDate?: boolean;    // default true — the whole date line
  showWeekday?: boolean; // default true — the day name within the date
}
```

(b) In the `PrinterObject` interface, add the field after `icon?`:

```ts
  icon?: PrinterIcon; // icon objects
  clock?: PrinterClockOptions; // clock objects
```

(c) Next to `sanitizeIcon` (after it), add:

```ts
function sanitizeClock(raw: unknown): PrinterClockOptions {
  const r = (raw ?? {}) as Record<string, unknown>;
  return {
    showDate: typeof r.showDate === "boolean" ? r.showDate : true,
    showWeekday: typeof r.showWeekday === "boolean" ? r.showWeekday : true,
  };
}
```

(d) In `sanitizeObject`, add a clock branch **after** the `if (type === "icon")` block and **before** the `// widget singleton` fallback:

```ts
  if (type === "clock") {
    return {
      id, type: "clock", z, visible,
      ...sanitizeBox(o, WIDGET_BOX.clock),
      align: ALIGNS.includes(o.align as TextAlign) ? (o.align as TextAlign) : "center",
      clock: sanitizeClock(o.clock),
    };
  }
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run printer-layout`
Expected: PASS. Then `npx tsc --noEmit` — clean.

- [ ] **Step 5: Commit**

```bash
git add lib/printer-layout.ts lib/printer-layout.test.ts
git commit -m "feat(branding): clock display options (showDate/showWeekday/align) in the config model"
```

---

### Task 3: `topBarArrangement()` helper

**Files:**
- Modify: `lib/printer-layout.ts`
- Test: `lib/printer-layout.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { topBarArrangement } from "./printer-layout";

describe("topBarArrangement", () => {
  it("returns on-canvas boxes for logo (left), clock (center, compact), wifi (right)", () => {
    const bar = topBarArrangement();
    for (const key of ["logo", "clock", "wifi"] as const) {
      const b = bar[key];
      expect(b.x! >= 0 && b.y! >= 0).toBe(true);
      expect(b.x! + b.w! <= 1.0001 && b.y! + b.h! <= 1.0001).toBe(true);
    }
    // logo left of wifi; clock between them
    expect(bar.logo.x! < bar.clock.x!).toBe(true);
    expect(bar.clock.x! < bar.wifi.x!).toBe(true);
    // clock is compact (date hidden) in the bar
    expect(bar.clock.clock).toEqual({ showDate: false, showWeekday: true });
    expect(bar.clock.align).toBe("center");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run printer-layout`
Expected: FAIL — `topBarArrangement` not exported.

- [ ] **Step 3: Implement**

In `lib/printer-layout.ts` (after `createIconObject`), add:

```ts
/** Boxes + clock settings for a tidy top status row: logo (left) · clock (center, compact) · wifi (right). */
export function topBarArrangement(): Record<"logo" | "clock" | "wifi", Partial<PrinterObject>> {
  return {
    logo: { x: 0.04, y: 0.04, w: 0.3, h: 0.1 },
    clock: { x: 0.4, y: 0.045, w: 0.2, h: 0.09, align: "center", clock: { showDate: false, showWeekday: true } },
    wifi: { x: 0.86, y: 0.05, w: 0.1, h: 0.06 },
  };
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run printer-layout`
Expected: PASS. Then `npx tsc --noEmit` — clean.

- [ ] **Step 5: Commit**

```bash
git add lib/printer-layout.ts lib/printer-layout.test.ts
git commit -m "feat(branding): topBarArrangement() preset helper"
```

---

### Task 4: `PrinterClock` honors showDate / showWeekday / align

**Files:**
- Modify: `components/device-preview/printer-clock.tsx`

- [ ] **Step 1: Add the props + conditional formatting**

Replace the `PrinterClock` signature and the date computation/render. New signature + body changes:

```tsx
export function PrinterClock({
  timezone,
  hour24 = false,
  size = 84,
  showDate = true,
  showWeekday = true,
  align = "center",
}: {
  timezone: string;
  hour24?: boolean;
  size?: number;
  showDate?: boolean;
  showWeekday?: boolean;
  align?: "left" | "center" | "right";
}) {
```

In the `if (now)` block, change the `date = ...` line to drop the weekday when `showWeekday` is false:

```ts
      date = now.toLocaleDateString([], {
        ...(showWeekday ? { weekday: "long" as const } : {}),
        month: "long",
        day: "numeric",
        timeZone: timezone,
      });
```

In the returned JSX, set the root alignment to `align` and gate the date line on `showDate`:

```tsx
  return (
    <div style={{ textAlign: align }} suppressHydrationWarning>
      <div
        style={{
          fontSize: cq(size),
          fontWeight: 700,
          letterSpacing: "-1.5px",
          lineHeight: 1,
          fontVariantNumeric: "tabular-nums",
          color: "var(--k-fg)",
        }}
      >
        {time}
      </div>
      {showDate && date && (
        <div style={{ fontSize: cq(size * 0.26), fontWeight: 500, color: "var(--k-muted)", marginTop: cq(size * 0.14) }}>
          {date}
        </div>
      )}
    </div>
  );
```

- [ ] **Step 2: Verify it type-checks + builds**

Run: `npx tsc --noEmit && npm run build`
Expected: clean (callers still pass the old props; new props are optional). The `ClockObject` caller is updated in Task 5 to pass them.

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/printer-clock.tsx
git commit -m "feat(branding): PrinterClock supports hide-date, hide-weekday, and alignment"
```

---

### Task 5: `ClockObject` passes clock options + align through

**Files:**
- Modify: `components/device-preview/printer-preview.tsx`

- [ ] **Step 1: Update `ClockObject`**

Replace the `ClockObject` function body so it derives alignment and forwards the new props:

```tsx
function ClockObject({
  object,
  timezone,
  clock24h,
}: {
  object: PrinterObject;
  timezone: string;
  clock24h: boolean;
}) {
  const timeFont = object.h * 720 * 0.5; // time font ~ half the box height
  const align = object.align ?? "center";
  const justify = align === "left" ? "flex-start" : align === "right" ? "flex-end" : "center";
  return (
    <div style={{ width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: justify, overflow: "hidden" }}>
      <PrinterClock
        timezone={timezone}
        hour24={clock24h}
        size={timeFont}
        showDate={object.clock?.showDate ?? true}
        showWeekday={object.clock?.showWeekday ?? true}
        align={align}
      />
    </div>
  );
}
```

- [ ] **Step 2: Verify build + smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: clean.

Smoke (dev server, `/tenant/branding`, idle screen): the clock looks unchanged at defaults (date + weekday, centered). (Editing controls land in Task 7.)

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/printer-preview.tsx
git commit -m "feat(branding): clock object honors per-screen display options + align"
```

---

### Task 6: `insertTopBar()` editor action

**Files:**
- Modify: `components/device-preview/printer-editor/use-printer-editor.ts`

- [ ] **Step 1: Import the helper**

Add `topBarArrangement` to the import from `@/lib/printer-layout`:

```ts
import {
  createTextObject,
  createIconObject,
  seededScreen,
  topBarArrangement,
  MAX_CUSTOM,
  type PrinterObject,
  type PrinterConfig,
  type PrinterScreen,
} from "@/lib/printer-layout";
```

- [ ] **Step 2: Add the action**

After `resetLayout()` (before `endInteraction`), add:

```ts
  /** Snap logo/clock/wifi into a tidy top row on the active screen (creating any that are absent). */
  function insertTopBar() {
    if (disabled) return;
    const preset = topBarArrangement();
    const next = [...objects];
    let z = next.reduce((m, o) => Math.max(m, o.z), 0);
    (["logo", "clock", "wifi"] as const).forEach((type) => {
      z += 1;
      const patch = preset[type];
      const idx = next.findIndex((o) => o.type === type);
      if (idx >= 0) {
        next[idx] = { ...next[idx], ...patch, visible: true, z };
      } else {
        next.push({ id: type, type, x: 0.04, y: 0.04, w: 0.1, h: 0.1, visible: true, z, ...patch } as PrinterObject);
      }
    });
    setObjects(next);
  }
```

- [ ] **Step 3: Add to the interface + return**

In the `PrinterEditor` interface, after `resetLayout: () => void;` add:

```ts
  insertTopBar: () => void;
```

In the returned object (after `resetLayout,`) add `insertTopBar,`.

- [ ] **Step 4: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: clean.

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/printer-editor/use-printer-editor.ts
git commit -m "feat(branding): insertTopBar() editor action"
```

---

### Task 7: Controls — clock options UI + "Insert top bar" button

**Files:**
- Modify: `components/device-preview/printer-editor/printer-controls.tsx`

- [ ] **Step 1: Add the "Insert top bar" button**

Add the `PanelTop` icon to the lucide import (line 4):

```ts
import { AlignCenter, AlignLeft, AlignRight, Eye, EyeOff, PanelTop, Plus, RotateCcw, Trash2, Wifi } from "lucide-react";
```

In `PrinterControls`, insert this button between the Objects card (`</div>` at the end of the `rounded-xl border p-3` block) and the `{selected && <Properties .../>}` line:

```tsx
      <button
        type="button"
        disabled={disabled}
        onClick={editor.insertTopBar}
        className="flex w-full items-center justify-center gap-2 rounded-lg border py-2 text-sm font-medium text-muted-foreground transition-colors hover:bg-accent hover:text-foreground disabled:opacity-50"
      >
        <PanelTop className="size-4" /> Insert top bar
      </button>
```

- [ ] **Step 2: Add clock Align + Show date + Show weekday to the clock Properties**

In `Properties`, inside the `object.type === "clock"` block, after the timezone+24h `<div className="grid gap-3 sm:grid-cols-2">…</div>`, add (still inside the clock block):

```tsx
          <div className="space-y-1">
            <Label className="text-[10px] text-muted-foreground">Align</Label>
            <div className="flex gap-1">
              {([["left", AlignLeft], ["center", AlignCenter], ["right", AlignRight]] as [TextAlign, typeof AlignLeft][]).map(([a, Icon]) => (
                <button
                  key={a}
                  type="button"
                  disabled={disabled}
                  onClick={() => set({ align: a })}
                  aria-label={`Align ${a}`}
                  className={cn("flex h-8 flex-1 items-center justify-center rounded-md border transition-colors disabled:opacity-50", (object.align ?? "center") === a ? "border-foreground bg-foreground text-background" : "hover:bg-accent")}
                >
                  <Icon className="size-4" />
                </button>
              ))}
            </div>
          </div>
          <div className="flex items-center justify-between">
            <Label htmlFor="clock-show-date" className="text-xs text-muted-foreground">Show date</Label>
            <Switch
              id="clock-show-date"
              checked={object.clock?.showDate ?? true}
              onCheckedChange={(v) => set({ clock: { ...(object.clock ?? {}), showDate: v } })}
              disabled={disabled}
            />
          </div>
          <div className="flex items-center justify-between">
            <Label htmlFor="clock-show-weekday" className="text-xs text-muted-foreground">Show weekday</Label>
            <Switch
              id="clock-show-weekday"
              checked={object.clock?.showWeekday ?? true}
              onCheckedChange={(v) => set({ clock: { ...(object.clock ?? {}), showWeekday: v } })}
              disabled={disabled || !(object.clock?.showDate ?? true)}
            />
          </div>
```

(`set`, `cn`, `TextAlign`, `Switch`, `Label`, and the Align icons are already imported/defined in this file.)

- [ ] **Step 3: Verify build + smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: clean.

Smoke (dev server, `/tenant/branding`):
- Select the **clock** → an **Align** row + **Show date** / **Show weekday** toggles appear. Turning Show date off hides the date line (and disables Show weekday); Show weekday off shows "June 14"; Align left pushes the clock text to the left.
- Click **Insert top bar** → logo/clock/wifi snap to a top row (logo left, compact clock center, wifi right).
- **Save** → reload → the clock options + top-bar arrangement persist.
- A fresh tenant's idle screen shows **no** "Lane 1"/tagline labels.

- [ ] **Step 4: Commit**

```bash
git add components/device-preview/printer-editor/printer-controls.tsx
git commit -m "feat(branding): clock display options UI + Insert top bar button"
```

---

## Self-Review Notes

- **Spec coverage:** cleaner default → Task 1; clock options (data/normalize) → Task 2, (render) → Tasks 4–5, (UI) → Task 7; top-bar preset (helper) → Task 3, (action) → Task 6, (button) → Task 7. All spec sections map to a task.
- **Type consistency:** `PrinterClockOptions` + `clock?` (Task 2) are consumed by `topBarArrangement()` (Task 3), `ClockObject` (Task 5), `insertTopBar()` (Task 6), and the clock Properties UI (Task 7). `PrinterClock`'s new optional props (Task 4) match the call in Task 5. `insertTopBar` is added to the `PrinterEditor` interface (Task 6) and consumed in Task 7.
- **No DB migration:** the `clock` field lives in the v3 `PrinterConfig` jsonb; `normalizePrinterConfig` (Task 2) defaults it, so existing tenants are unaffected and render unchanged.
- **No placeholders:** every code step shows the exact code; the `topBarArrangement` boxes are concrete (and noted tunable in the spec; Task 7 smoke confirms the look).
- **Back-compat:** an existing clock object with no `clock`/`align` renders identically to today (defaults `showDate:true`, `showWeekday:true`, `align:"center"`).
