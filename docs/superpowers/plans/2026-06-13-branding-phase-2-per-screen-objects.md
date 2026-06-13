# Branding Phase 2 — Per-Screen Object Model + Icon System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make every one of the 7 kiosk screens fully editable (object-based, like the idle screen is today) and add a customizable **icon** widget — delivering the editable Success ✓ icon — with a safe v2→v3 data migration and zero visual change for tenants who don't touch the new screens.

**Architecture:** Generalize the single-screen `KioskLayout` (v2) to a `KioskConfig` (v3) holding a `ScreenLayout` per screen, with clock/wifi/timezone promoted to shared top-level config. Lift the bespoke per-screen JSX in `kiosk-preview.tsx` into per-widget renderers so `KioskPreview` and `KioskStage` both render `screen.objects.map(ObjectVisual)`. New `icon` objects render a curated lucide glyph or an uploaded R2 image. `useKioskEditor` operates on the active screen's objects. Persistence adds a `tenant_settings.kioskScreens` jsonb column (keeping `kioskLayout` one release for rollback) with a backfill migration; all stored data flows through a `normalizeKioskConfig` that can never throw.

**Tech Stack:** Next.js 16 (App Router), React 19, TypeScript strict, Tailwind v4, unified `radix-ui` package (same import style as `components/ui/tabs.tsx`), lucide-react, Drizzle ORM over neon-http, Cloudflare R2 (`lib/storage.ts`), vitest 3 (pure-function tests; UI tasks verify via `npx tsc --noEmit` + `npm run build` + a dev-server/Playwright smoke check, matching Phase 1).

**Spec:** `docs/superpowers/specs/2026-06-13-branding-page-improvements-design.md` (Phase 2 section, lines 130–283).

**Phase 1 (shipped) baseline this builds on:** `cq` and `kioskRootStyle` are exported from `kiosk-preview.tsx`; `InlineTextEditor` lives in `kiosk-stage.tsx`; `KioskEditor` exposes `isDragging()`; `branding-editor.tsx` is a two-pane accordion + zoom + `PreviewCarousel` over all 7 screens, with a placeholder in the "{screen} content" accordion item for non-idle screens (Task 11 removes that placeholder).

---

## File Structure

| File | Create/Modify | Responsibility |
|---|---|---|
| `lib/kiosk-layout.ts` | Modify | v3 types (`KioskObjectType` expansion, `KioskIcon`, `ScreenLayout`, `KioskConfig`), `KIOSK_SCREENS`, `ICON_PRESETS` allowlist, `seededScreen`, `createIconObject`, `migrateV2ToConfig`, `normalizeKioskConfig`. Keep all v2 exports for the migration. |
| `lib/kiosk-layout.test.ts` | Create | Vitest for `seededScreen`, `migrateV2ToConfig`, `normalizeKioskConfig`, `createIconObject`. |
| `lib/kiosk-icons.ts` | Create | Client-only map of `IconPreset` name → lucide component + `resolveIconComponent`. |
| `lib/kiosk-icons.test.ts` | Create | Every preset resolves; unknown falls back. |
| `components/ui/popover.tsx` | Create | shadcn Popover over `radix-ui` (radix-nova), for the icon picker. |
| `components/device-preview/kiosk-preview.tsx` | Modify | `ObjectVisual` gains widget cases (`icon`/`qr`/`spinner`/`countdown`/`pairingCode`/`steps`); per-screen `*Screen` JSX lifted into per-widget renderers; `KioskPreview` renders `screen.objects.map(ObjectVisual)`. |
| `components/device-preview/kiosk-editor/use-kiosk-editor.ts` | Modify | Operate on `config.screens[screen]`; add `addIcon`; per-screen `resetLayout`; write shared clock/wifi to top level. |
| `components/device-preview/kiosk-editor/kiosk-stage.tsx` | Modify | Render the active screen's objects via `ObjectVisual`; inline text edit unchanged. |
| `components/device-preview/kiosk-editor/kiosk-controls.tsx` | Modify | Type-aware Properties: icon picker popover (Library/Upload tabs) + tint + circle; `Add icon` button. |
| `components/device-preview/kiosk-icon-picker.tsx` | Create | The icon picker popover content (Library grid + Upload). |
| `lib/db/schema.ts` | Modify | Add `tenantSettings.kioskScreens jsonb`. |
| `lib/db/migrations/*` | Create (generated) | `db:generate` SQL adding the column + a backfill. |
| `app/(tenant)/tenant/branding/actions.ts` | Modify | `saveBranding` accepts the full v3 config + per-object icon uploads; presign/cleanup icon keys. |
| `lib/data.ts` | Modify | `getTenantBranding` returns a normalized `KioskConfig` and presigns all icon keys across screens. |
| `lib/storage.ts` | Modify | Add `iconStorageKey(orgId, assetId)`. |
| `components/branding-editor.tsx` | Modify | Drive the editor from `KioskConfig` + active `screen`; editable stage for every screen; submit the config + icon files. |

Each task produces a self-contained, committable change.

---

## Shared type contract (used across tasks — defined in Task 1)

```ts
// lib/kiosk-layout.ts (v3 additions)

export const KIOSK_SCREENS = ["idle", "processing", "qr", "sent", "error", "paused", "setup"] as const;
export type KioskScreen = (typeof KIOSK_SCREENS)[number];

export const OBJECT_TYPES = [
  "text", "logo", "clock", "wifi",           // existing
  "icon",                                     // new — curated glyph or uploaded image
  "qr", "spinner", "countdown",               // new fixed-render widgets
  "pairingCode", "steps",
] as const;
export type KioskObjectType = (typeof OBJECT_TYPES)[number];

/** Types the user can add/duplicate (everything else is a per-screen singleton). */
export const ADDABLE_TYPES = ["text", "icon"] as const;
/** Singleton widgets: ≤1 per screen, hideable, not deletable, not user-addable. */
export const WIDGET_TYPES = ["logo", "clock", "wifi", "qr", "spinner", "countdown", "pairingCode", "steps"] as const;
export type WidgetType = (typeof WIDGET_TYPES)[number];

export const ICON_PRESETS = [
  "check", "check-circle", "heart", "star", "gift", "mail", "thumbs-up", "smile",
  "clock", "bell", "alert-triangle", "wifi-off", "sparkles", "party-popper",
  "badge-check", "coffee",
] as const;
export type IconPreset = (typeof ICON_PRESETS)[number];
export const DEFAULT_ICON_PRESET: IconPreset = "check";

export type IconTint = "accent" | "muted" | "warn" | "none";

export interface KioskIcon {
  source: "preset" | "upload";
  preset?: IconPreset;   // when source = "preset"
  url?: string;          // R2 object key when source = "upload" (presigned on read)
  tint?: IconTint;       // default "accent"
  circle?: boolean;      // filled circular background (the Sent ✓ look)
}

// KioskObject (v3) = the existing v2 KioskObject plus an optional icon:
export interface KioskObject {
  id: string;
  type: KioskObjectType;
  x: number; y: number; w: number; h: number; // fractions 0..1 on the 720² canvas
  visible: boolean;
  z: number;
  text?: string; fontSize?: number; align?: TextAlign; // text
  icon?: KioskIcon;                                     // icon
}

export interface ScreenLayout { objects: KioskObject[]; }

export interface KioskConfig {
  version: 3;
  clockTimezone: string; // shared (clock widget)
  clock24h: boolean;     // shared
  wifiLevel: number;     // shared (wifi widget), 0..4
  screens: Record<KioskScreen, ScreenLayout>;
}
```

> **Renderer contract change (Task 5):** `ObjectVisual` signature changes from
> `{ object, brand, layout }` to `{ object, brand, config }` where `config: KioskConfig`
> (widgets read `config.clockTimezone`/`clock24h`/`wifiLevel`). Every later task uses this signature.

---

### Task 1: v3 data model — types, screen list, icon allowlist, `seededScreen`, `createIconObject`

**Files:**
- Modify: `lib/kiosk-layout.ts`
- Test: `lib/kiosk-layout.test.ts` (create)

This task adds the v3 types **alongside** the existing v2 types (keep `KioskLayout`, `defaultLayout`, `normalizeKioskLayout`, `seededText`, `FIXED_DEFAULTS`, `createTextObject`, `objectLabel`, all constants — Task 2's migration needs them). It also adds `seededScreen` (the per-screen default builder) and `createIconObject`.

- [ ] **Step 1: Write the failing test**

Create `lib/kiosk-layout.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import {
  KIOSK_SCREENS,
  ICON_PRESETS,
  DEFAULT_ICON_PRESET,
  MIN_BOX_OK, // helper below
  seededScreen,
  createIconObject,
  type KioskObject,
} from "./kiosk-layout";

// A box is valid if it sits on the canvas and meets the min size.
function boxesValid(objects: KioskObject[]): boolean {
  return objects.every(
    (o) =>
      o.x >= 0 && o.y >= 0 && o.w > 0 && o.h > 0 &&
      o.x + o.w <= 1.0001 && o.y + o.h <= 1.0001 &&
      typeof o.z === "number" && typeof o.visible === "boolean",
  );
}

describe("seededScreen", () => {
  it("produces a non-empty, on-canvas layout for every screen", () => {
    for (const screen of KIOSK_SCREENS) {
      const { objects } = seededScreen(screen);
      expect(objects.length).toBeGreaterThan(0);
      expect(boxesValid(objects)).toBe(true);
      // ids are unique within a screen
      expect(new Set(objects.map((o) => o.id)).size).toBe(objects.length);
    }
  });

  it("seeds the sent screen with an accent circle check icon", () => {
    const sent = seededScreen("sent").objects.find((o) => o.type === "icon");
    expect(sent).toBeDefined();
    expect(sent!.icon).toMatchObject({ source: "preset", preset: "check", circle: true, tint: "accent" });
  });

  it("seeds the error screen with a warn wifi-off icon", () => {
    const err = seededScreen("error").objects.find((o) => o.type === "icon");
    expect(err!.icon).toMatchObject({ source: "preset", preset: "wifi-off", tint: "warn" });
  });

  it("seeds idle with the existing logo/clock/wifi widgets and two text objects", () => {
    const idle = seededScreen("idle").objects;
    expect(idle.some((o) => o.type === "logo")).toBe(true);
    expect(idle.some((o) => o.type === "clock")).toBe(true);
    expect(idle.some((o) => o.type === "wifi")).toBe(true);
    expect(idle.filter((o) => o.type === "text").length).toBe(2);
  });
});

describe("createIconObject", () => {
  it("creates a centered preset icon on top", () => {
    const o = createIconObject(5);
    expect(o.type).toBe("icon");
    expect(o.z).toBe(5);
    expect(o.icon).toMatchObject({ source: "preset", preset: DEFAULT_ICON_PRESET, tint: "accent" });
    expect(o.id.startsWith("icon-")).toBe(true);
  });
});

describe("allowlist", () => {
  it("DEFAULT_ICON_PRESET is in ICON_PRESETS", () => {
    expect((ICON_PRESETS as readonly string[]).includes(DEFAULT_ICON_PRESET)).toBe(true);
  });
});
```

> Note: `MIN_BOX_OK` is imported only to keep the import list honest if you add helpers; if you don't export it, remove it from the import. (It is not asserted directly.)

- [ ] **Step 2: Run the test to verify it fails**

Run: `npx vitest run kiosk-layout`
Expected: FAIL — `seededScreen`, `createIconObject`, `KIOSK_SCREENS` not exported.

- [ ] **Step 3: Implement the v3 types + builders**

In `lib/kiosk-layout.ts`:

(a) Replace the `OBJECT_TYPES`/`KioskObjectType` block (lines 10–14) and `TYPE_LABEL` (lines 16–21) with:

```ts
export const KIOSK_SCREENS = ["idle", "processing", "qr", "sent", "error", "paused", "setup"] as const;
export type KioskScreen = (typeof KIOSK_SCREENS)[number];

export const OBJECT_TYPES = [
  "text", "logo", "clock", "wifi",
  "icon",
  "qr", "spinner", "countdown", "pairingCode", "steps",
] as const;
export type KioskObjectType = (typeof OBJECT_TYPES)[number];

// Legacy v2 fixed widgets (idle screen). Kept for v2 normalize + migration.
export const FIXED_TYPES = ["logo", "clock", "wifi"] as const;
export type FixedType = (typeof FIXED_TYPES)[number];

// v3 singleton widgets: ≤1 per screen; hideable; not deletable; not user-addable.
export const WIDGET_TYPES = ["logo", "clock", "wifi", "qr", "spinner", "countdown", "pairingCode", "steps"] as const;
export type WidgetType = (typeof WIDGET_TYPES)[number];

// v3 user-addable/duplicable types.
export const ADDABLE_TYPES = ["text", "icon"] as const;

export const TYPE_LABEL: Record<KioskObjectType, string> = {
  text: "Text",
  logo: "Logo",
  clock: "Clock",
  wifi: "Wi-Fi signal",
  icon: "Icon",
  qr: "QR code",
  spinner: "Spinner",
  countdown: "Countdown",
  pairingCode: "Pairing code",
  steps: "Steps",
};

export const ICON_PRESETS = [
  "check", "check-circle", "heart", "star", "gift", "mail", "thumbs-up", "smile",
  "clock", "bell", "alert-triangle", "wifi-off", "sparkles", "party-popper",
  "badge-check", "coffee",
] as const;
export type IconPreset = (typeof ICON_PRESETS)[number];
export const DEFAULT_ICON_PRESET: IconPreset = "check";
export type IconTint = "accent" | "muted" | "warn" | "none";

export interface KioskIcon {
  source: "preset" | "upload";
  preset?: IconPreset;
  url?: string;
  tint?: IconTint;
  circle?: boolean;
}
```

(b) Add `icon?: KioskIcon;` to the `KioskObject` interface (after the `align?` line, currently line 36):

```ts
  align?: TextAlign;
  icon?: KioskIcon; // icon objects
```

(c) After the existing `KioskLayout` interface (line 45), add the v3 config types:

```ts
export interface ScreenLayout {
  objects: KioskObject[];
}

export interface KioskConfig {
  version: 3;
  clockTimezone: string;
  clock24h: boolean;
  wifiLevel: number; // 0..4
  screens: Record<KioskScreen, ScreenLayout>;
}
```

(d) After `createTextObject` (line 113), add `createIconObject` and the per-screen seed builder. The seed positions are **lifted from the existing per-screen JSX** in `kiosk-preview.tsx` (see line refs); reproduce them as objects so screens look identical to today:

```ts
/** A fresh custom icon object, centered, on top (`z`). */
export function createIconObject(z: number): KioskObject {
  const rand = typeof crypto !== "undefined" && crypto.randomUUID
    ? crypto.randomUUID().slice(0, 8)
    : Math.floor(Math.random() * 1e9).toString(36);
  return {
    id: `icon-${rand}`,
    type: "icon",
    x: 0.4, y: 0.4, w: 0.2, h: 0.2,
    visible: true,
    z,
    icon: { source: "preset", preset: DEFAULT_ICON_PRESET, tint: "accent", circle: false },
  };
}

// Internal helper: build an object with sane defaults.
function obj(o: Partial<KioskObject> & Pick<KioskObject, "id" | "type" | "x" | "y" | "w" | "h" | "z">): KioskObject {
  return { visible: true, ...o };
}

/**
 * Default object layout for a screen, reproducing today's hard-coded templates so
 * a tenant who never edits a screen sees no visual change. Positions/sizes are
 * lifted from the per-screen JSX in kiosk-preview.tsx; tune to match pixel-for-pixel
 * against the live render during the Task 5 smoke check.
 */
export function seededScreen(screen: KioskScreen): ScreenLayout {
  switch (screen) {
    case "idle":
      // Reuse the v2 default objects verbatim (logo/clock/wifi + lane + tagline).
      return { objects: defaultLayout().objects };
    case "processing":
      // From ProcessingScreen (kiosk-preview.tsx:318–340): spinner + caption text.
      return {
        objects: [
          obj({ id: "spinner", type: "spinner", x: 0.42, y: 0.34, w: 0.16, h: 0.16, z: 0 }),
          obj({ id: "text-caption", type: "text", x: 0.15, y: 0.56, w: 0.7, h: 0.08, z: 1, text: "Preparing your receipt…", fontSize: 26, align: "center" }),
          obj({ id: "text-sub", type: "text", x: 0.2, y: 0.66, w: 0.6, h: 0.06, z: 2, text: "This only takes a moment", fontSize: 16, align: "center" }),
        ],
      };
    case "qr":
      // From ReceiptScreen (kiosk-preview.tsx:343–405): logo + heading + qr + caption + countdown.
      return {
        objects: [
          obj({ id: "logo", type: "logo", x: 0.34, y: 0.06, w: 0.32, h: 0.12, z: 0 }),
          obj({ id: "text-heading", type: "text", x: 0.1, y: 0.2, w: 0.8, h: 0.07, z: 1, text: "Scan to get your receipt", fontSize: 24, align: "center" }),
          obj({ id: "qr", type: "qr", x: 0.32, y: 0.3, w: 0.36, h: 0.36, z: 2 }),
          obj({ id: "text-hint", type: "text", x: 0.15, y: 0.7, w: 0.7, h: 0.06, z: 3, text: "Point your phone camera at the code", fontSize: 16, align: "center" }),
          obj({ id: "countdown", type: "countdown", x: 0.3, y: 0.8, w: 0.4, h: 0.1, z: 4 }),
        ],
      };
    case "sent":
      // From SentScreen (kiosk-preview.tsx:408–447): the hard-coded check SVG (421–423)
      // becomes an icon object; title/subtext/footer become text.
      return {
        objects: [
          obj({ id: "icon", type: "icon", x: 0.4, y: 0.22, w: 0.2, h: 0.2, z: 0, icon: { source: "preset", preset: "check", circle: true, tint: "accent" } }),
          obj({ id: "text-title", type: "text", x: 0.1, y: 0.48, w: 0.8, h: 0.08, z: 1, text: "Your receipt is on its way", fontSize: 26, align: "center" }),
          obj({ id: "text-sub", type: "text", x: 0.15, y: 0.58, w: 0.7, h: 0.06, z: 2, text: "Check your phone — all set. Thank you!", fontSize: 16, align: "center" }),
          obj({ id: "text-footer", type: "text", x: 0.2, y: 0.82, w: 0.6, h: 0.05, z: 3, text: "Returning to start…", fontSize: 14, align: "center" }),
        ],
      };
    case "error":
      // From ErrorScreen (kiosk-preview.tsx:450–489): wifi-off icon + headline + subtext + pill.
      return {
        objects: [
          obj({ id: "icon", type: "icon", x: 0.42, y: 0.22, w: 0.16, h: 0.16, z: 0, icon: { source: "preset", preset: "wifi-off", tint: "warn", circle: false } }),
          obj({ id: "text-title", type: "text", x: 0.1, y: 0.44, w: 0.8, h: 0.08, z: 1, text: "We couldn’t send your receipt", fontSize: 24, align: "center" }),
          obj({ id: "text-sub", type: "text", x: 0.15, y: 0.54, w: 0.7, h: 0.06, z: 2, text: "The device is offline right now.", fontSize: 16, align: "center" }),
          obj({ id: "text-pill", type: "text", x: 0.15, y: 0.72, w: 0.7, h: 0.08, z: 3, text: "Please ask a team member for a paper receipt", fontSize: 15, align: "center" }),
        ],
      };
    case "paused":
      // From PausedScreen (kiosk-preview.tsx:492–523): dimmed logo + text.
      return {
        objects: [
          obj({ id: "logo", type: "logo", x: 0.34, y: 0.22, w: 0.32, h: 0.16, z: 0 }),
          obj({ id: "text-title", type: "text", x: 0.1, y: 0.46, w: 0.8, h: 0.08, z: 1, text: "Currently unavailable", fontSize: 24, align: "center" }),
          obj({ id: "text-sub", type: "text", x: 0.15, y: 0.56, w: 0.7, h: 0.06, z: 2, text: "Digital receipts are paused at this register.", fontSize: 16, align: "center" }),
        ],
      };
    case "setup":
      // From SetupScreen (kiosk-preview.tsx:526–612): logo + heading + steps + pairingCode + qr.
      return {
        objects: [
          obj({ id: "logo", type: "logo", x: 0.34, y: 0.05, w: 0.32, h: 0.1, z: 0 }),
          obj({ id: "text-title", type: "text", x: 0.1, y: 0.18, w: 0.8, h: 0.07, z: 1, text: "Let’s pair this device", fontSize: 24, align: "center" }),
          obj({ id: "text-sub", type: "text", x: 0.15, y: 0.26, w: 0.7, h: 0.05, z: 2, text: "Claim it from your admin dashboard to start.", fontSize: 15, align: "center" }),
          obj({ id: "steps", type: "steps", x: 0.18, y: 0.34, w: 0.64, h: 0.28, z: 3 }),
          obj({ id: "pairingCode", type: "pairingCode", x: 0.25, y: 0.66, w: 0.3, h: 0.12, z: 4 }),
          obj({ id: "qr", type: "qr", x: 0.6, y: 0.66, w: 0.2, h: 0.2, z: 5 }),
        ],
      };
  }
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npx vitest run kiosk-layout`
Expected: PASS (all cases green). Then `npx tsc --noEmit` — expect no errors (existing v2 code still compiles because v2 types are untouched).

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-layout.ts lib/kiosk-layout.test.ts
git commit -m "feat(branding): v3 kiosk config types + per-screen seeded defaults"
```

---

### Task 2: v2→v3 migration + `normalizeKioskConfig`

**Files:**
- Modify: `lib/kiosk-layout.ts`
- Test: `lib/kiosk-layout.test.ts` (extend)

Mirror the robustness of `normalizeKioskLayout`: never throw, coerce any shape into a valid v3 config, fill missing screens from seeds, cap per-screen objects, validate icons, clamp geometry. A stored **v2** layout migrates into `screens.idle` with the rest seeded.

- [ ] **Step 1: Write the failing test (extend `kiosk-layout.test.ts`)**

```ts
import {
  migrateV2ToConfig,
  normalizeKioskConfig,
  defaultLayout,
  MAX_CUSTOM,
  KIOSK_SCREENS,
} from "./kiosk-layout";

describe("migrateV2ToConfig", () => {
  it("puts the v2 idle objects into screens.idle and seeds the other 6", () => {
    const v2 = defaultLayout();
    const cfg = migrateV2ToConfig(v2);
    expect(cfg.version).toBe(3);
    expect(cfg.clockTimezone).toBe(v2.clockTimezone);
    expect(cfg.wifiLevel).toBe(v2.wifiLevel);
    expect(cfg.screens.idle.objects.length).toBe(v2.objects.length);
    for (const s of KIOSK_SCREENS) {
      expect(cfg.screens[s].objects.length).toBeGreaterThan(0);
    }
  });
});

describe("normalizeKioskConfig", () => {
  it("returns a fully-seeded default for garbage input", () => {
    const cfg = normalizeKioskConfig(null);
    expect(cfg.version).toBe(3);
    for (const s of KIOSK_SCREENS) expect(cfg.screens[s].objects.length).toBeGreaterThan(0);
  });

  it("migrates a stored v2 layout (version: 2)", () => {
    const v2 = defaultLayout();
    const cfg = normalizeKioskConfig(v2);
    expect(cfg.version).toBe(3);
    expect(cfg.screens.idle.objects.some((o) => o.type === "text")).toBe(true);
  });

  it("fills a missing screen from its seed", () => {
    const cfg = normalizeKioskConfig({
      version: 3, clockTimezone: "UTC", clock24h: false, wifiLevel: 3,
      screens: { idle: { objects: [] } }, // others absent
    });
    expect(cfg.screens.sent.objects.length).toBeGreaterThan(0);
  });

  it("drops an unknown icon preset to the default and keeps the object", () => {
    const cfg = normalizeKioskConfig({
      version: 3, clockTimezone: "UTC", clock24h: false, wifiLevel: 3,
      screens: { idle: { objects: [
        { id: "i1", type: "icon", x: 0.4, y: 0.4, w: 0.2, h: 0.2, visible: true, z: 0,
          icon: { source: "preset", preset: "definitely-not-a-real-icon" } },
      ] } },
    });
    const icon = cfg.screens.idle.objects.find((o) => o.type === "icon");
    expect(icon!.icon!.preset).toBe("check");
  });

  it("caps addable (text+icon) objects per screen at MAX_CUSTOM", () => {
    const many = Array.from({ length: MAX_CUSTOM + 10 }, (_, i) => ({
      id: `t${i}`, type: "text", x: 0.1, y: 0.1, w: 0.3, h: 0.1, visible: true, z: i, text: `t${i}`,
    }));
    const cfg = normalizeKioskConfig({
      version: 3, clockTimezone: "UTC", clock24h: false, wifiLevel: 3,
      screens: { idle: { objects: many } },
    });
    const addable = cfg.screens.idle.objects.filter((o) => o.type === "text" || o.type === "icon");
    expect(addable.length).toBeLessThanOrEqual(MAX_CUSTOM);
  });

  it("clamps wifiLevel and out-of-range geometry", () => {
    const cfg = normalizeKioskConfig({
      version: 3, clockTimezone: "Nowhere/Nope", clock24h: "yes", wifiLevel: 99,
      screens: { idle: { objects: [
        { id: "t", type: "text", x: 5, y: -3, w: 9, h: 9, visible: true, z: 0, text: "x" },
      ] } },
    });
    expect(cfg.wifiLevel).toBe(4);
    expect(cfg.clockTimezone).toBe("UTC"); // invalid tz → UTC
    const t = cfg.screens.idle.objects.find((o) => o.id === "t")!;
    expect(t.x).toBeGreaterThanOrEqual(0);
    expect(t.x + t.w).toBeLessThanOrEqual(1.0001);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run kiosk-layout`
Expected: FAIL — `migrateV2ToConfig`/`normalizeKioskConfig` not exported.

- [ ] **Step 3: Implement migration + normalize**

Append to `lib/kiosk-layout.ts`. Reuse the existing module-private `clamp`, `num`, `sanitizeBox`, `ALIGNS`, `DEFAULT_FONT`, and `isValidTimezone` import:

```ts
const ICON_TINTS: IconTint[] = ["accent", "muted", "warn", "none"];

function sanitizeIcon(raw: unknown): KioskIcon {
  const r = (raw ?? {}) as Record<string, unknown>;
  const source = r.source === "upload" ? "upload" : "preset";
  const tint = ICON_TINTS.includes(r.tint as IconTint) ? (r.tint as IconTint) : "accent";
  const circle = typeof r.circle === "boolean" ? r.circle : false;
  if (source === "upload" && typeof r.url === "string" && r.url) {
    return { source: "upload", url: r.url, tint, circle };
  }
  const preset = (ICON_PRESETS as readonly string[]).includes(r.preset as string)
    ? (r.preset as IconPreset)
    : DEFAULT_ICON_PRESET;
  return { source: "preset", preset, tint, circle };
}

/** Default box for a widget singleton (used when a stored object is malformed). */
const WIDGET_BOX: Record<WidgetType, Pick<KioskObject, "x" | "y" | "w" | "h">> = {
  logo: { x: 0.34, y: 0.22, w: 0.32, h: 0.16 },
  clock: { x: 0.25, y: 0.52, w: 0.5, h: 0.18 },
  wifi: { x: 0.82, y: 0.04, w: 0.1, h: 0.06 },
  qr: { x: 0.32, y: 0.3, w: 0.36, h: 0.36 },
  spinner: { x: 0.42, y: 0.34, w: 0.16, h: 0.16 },
  countdown: { x: 0.3, y: 0.8, w: 0.4, h: 0.1 },
  pairingCode: { x: 0.25, y: 0.66, w: 0.3, h: 0.12 },
  steps: { x: 0.18, y: 0.34, w: 0.64, h: 0.28 },
};

/** Coerce one stored object into a valid KioskObject of a known type, or null to drop it. */
function sanitizeObject(raw: unknown, fallbackZ: number): KioskObject | null {
  const o = (raw ?? {}) as Record<string, unknown>;
  const type = o.type;
  if (!(OBJECT_TYPES as readonly string[]).includes(type as string)) return null;
  const z = typeof o.z === "number" && Number.isFinite(o.z) ? o.z : fallbackZ;
  const visible = typeof o.visible === "boolean" ? o.visible : true;
  const id = typeof o.id === "string" && o.id ? o.id : `${String(type)}-${fallbackZ}`;

  if (type === "text") {
    if (typeof o.text !== "string" || o.text.trim() === "") return null;
    return {
      id, type: "text", z, visible,
      ...sanitizeBox(o, { x: 0.35, y: 0.45, w: 0.3, h: 0.1 }),
      text: o.text.slice(0, MAX_TEXT_LEN),
      fontSize: clamp(num(o.fontSize, DEFAULT_FONT.text), FONT_MIN, FONT_MAX),
      align: ALIGNS.includes(o.align as TextAlign) ? (o.align as TextAlign) : "center",
    };
  }
  if (type === "icon") {
    return {
      id, type: "icon", z, visible,
      ...sanitizeBox(o, { x: 0.4, y: 0.4, w: 0.2, h: 0.2 }),
      icon: sanitizeIcon(o.icon),
    };
  }
  // widget singleton
  const wt = type as WidgetType;
  return {
    id, type: wt, z, visible,
    ...sanitizeBox(o, WIDGET_BOX[wt]),
  };
}

/** Normalize one screen's objects: ≥0 widget singletons (deduped) + capped addables. */
function sanitizeScreen(raw: unknown, screen: KioskScreen): ScreenLayout {
  const r = (raw ?? {}) as { objects?: unknown };
  if (!Array.isArray(r.objects)) return seededScreen(screen);
  const list = r.objects as unknown[];

  const out: KioskObject[] = [];
  const seenWidget = new Set<string>();
  let addable = 0;
  let zNext = 0;
  for (const item of list) {
    const o = sanitizeObject(item, zNext++);
    if (!o) continue;
    if (o.type === "text" || o.type === "icon") {
      if (addable >= MAX_CUSTOM) continue;
      addable++;
    } else {
      if (seenWidget.has(o.type)) continue; // one of each widget per screen
      seenWidget.add(o.type);
    }
    out.push(o);
  }
  // If a screen was emptied to nothing, fall back to its seed so it isn't blank.
  return { objects: out.length ? out : seededScreen(screen).objects };
}

/** Migrate a v2 KioskLayout into a v3 config: idle = its objects, others seeded. */
export function migrateV2ToConfig(layout: KioskLayout): KioskConfig {
  const screens = {} as Record<KioskScreen, ScreenLayout>;
  for (const s of KIOSK_SCREENS) {
    screens[s] = s === "idle" ? { objects: layout.objects } : seededScreen(s);
  }
  return {
    version: 3,
    clockTimezone: layout.clockTimezone,
    clock24h: layout.clock24h,
    wifiLevel: layout.wifiLevel,
    screens,
  };
}

/**
 * Coerce arbitrary stored data into a valid v3 KioskConfig. Accepts v3 directly,
 * migrates v2, and resets anything else to the fully-seeded default. Never throws.
 */
export function normalizeKioskConfig(raw: unknown): KioskConfig {
  const r = raw as { version?: unknown } | null;

  // v2 stored layout → migrate (run it through the v2 normalizer first for safety).
  if (r && typeof r === "object" && r.version === 2) {
    return migrateV2ToConfig(normalizeKioskLayout(r));
  }

  // Default fully-seeded config (also the v1/garbage fallback).
  const seededAll = (): KioskConfig => {
    const screens = {} as Record<KioskScreen, ScreenLayout>;
    for (const s of KIOSK_SCREENS) screens[s] = seededScreen(s);
    return { version: 3, clockTimezone: "UTC", clock24h: false, wifiLevel: 3, screens };
  };

  if (!r || typeof r !== "object" || r.version !== 3) return seededAll();

  const cfg = r as Record<string, unknown>;
  const rawScreens = (cfg.screens ?? {}) as Record<string, unknown>;
  const screens = {} as Record<KioskScreen, ScreenLayout>;
  for (const s of KIOSK_SCREENS) screens[s] = sanitizeScreen(rawScreens[s], s);

  const tz = typeof cfg.clockTimezone === "string" && isValidTimezone(cfg.clockTimezone)
    ? cfg.clockTimezone
    : "UTC";
  return {
    version: 3,
    clockTimezone: tz,
    clock24h: typeof cfg.clock24h === "boolean" ? cfg.clock24h : false,
    wifiLevel: clamp(Math.round(num(cfg.wifiLevel, 3)), 0, 4),
    screens,
  };
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run kiosk-layout`
Expected: PASS. Then `npx tsc --noEmit` — no errors.

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-layout.ts lib/kiosk-layout.test.ts
git commit -m "feat(branding): v2→v3 migration + normalizeKioskConfig"
```

---

### Task 3: Icon allowlist → lucide component map (`lib/kiosk-icons.ts`)

**Files:**
- Create: `lib/kiosk-icons.ts`
- Test: `lib/kiosk-icons.test.ts`

The map lives in its own client module (per the project gotcha: never pass lucide component functions across the server→client edge — the editor/preview resolve names→components **client-side**).

- [ ] **Step 1: Write the failing test**

Create `lib/kiosk-icons.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { ICON_PRESETS, DEFAULT_ICON_PRESET } from "./kiosk-layout";
import { resolveIconComponent, ICON_COMPONENTS } from "./kiosk-icons";

describe("kiosk-icons", () => {
  it("maps every preset to a component", () => {
    for (const name of ICON_PRESETS) {
      expect(typeof ICON_COMPONENTS[name]).toBe("function");
    }
  });
  it("falls back to the default preset's component for unknown names", () => {
    expect(resolveIconComponent("nope" as never)).toBe(ICON_COMPONENTS[DEFAULT_ICON_PRESET]);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run kiosk-icons`
Expected: FAIL — `Cannot find module './kiosk-icons'`.

- [ ] **Step 3: Implement**

Create `lib/kiosk-icons.ts`:

```ts
import {
  Check, CheckCircle2, Heart, Star, Gift, Mail, ThumbsUp, Smile, Clock, Bell,
  AlertTriangle, WifiOff, Sparkles, PartyPopper, BadgeCheck, Coffee,
  type LucideIcon,
} from "lucide-react";
import { ICON_PRESETS, DEFAULT_ICON_PRESET, type IconPreset } from "./kiosk-layout";

export const ICON_COMPONENTS: Record<IconPreset, LucideIcon> = {
  "check": Check,
  "check-circle": CheckCircle2,
  "heart": Heart,
  "star": Star,
  "gift": Gift,
  "mail": Mail,
  "thumbs-up": ThumbsUp,
  "smile": Smile,
  "clock": Clock,
  "bell": Bell,
  "alert-triangle": AlertTriangle,
  "wifi-off": WifiOff,
  "sparkles": Sparkles,
  "party-popper": PartyPopper,
  "badge-check": BadgeCheck,
  "coffee": Coffee,
};

/** Resolve a stored preset name to a lucide component, defaulting safely. */
export function resolveIconComponent(name: string | undefined): LucideIcon {
  if (name && (ICON_PRESETS as readonly string[]).includes(name)) {
    return ICON_COMPONENTS[name as IconPreset];
  }
  return ICON_COMPONENTS[DEFAULT_ICON_PRESET];
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run kiosk-icons`
Expected: PASS. (If any lucide name above doesn't exist in the installed `lucide-react`, `tsc` will flag the import — swap to the correct export name and update the `ICON_PRESETS` entry + `ICON_COMPONENTS` key together.)

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-icons.ts lib/kiosk-icons.test.ts
git commit -m "feat(branding): curated lucide icon allowlist + resolver"
```

---

### Task 4: Popover primitive (`components/ui/popover.tsx`)

**Files:**
- Create: `components/ui/popover.tsx`

The icon picker uses a popover; the repo has `tabs.tsx`/`dialog.tsx` but no popover. Add one over the unified `radix-ui` package (same style as `accordion.tsx`/`slider.tsx` from Phase 1).

- [ ] **Step 1: Create the component**

Create `components/ui/popover.tsx`:

```tsx
"use client";

import * as React from "react";
import { Popover as PopoverPrimitive } from "radix-ui";

import { cn } from "@/lib/utils";

function Popover(props: React.ComponentProps<typeof PopoverPrimitive.Root>) {
  return <PopoverPrimitive.Root data-slot="popover" {...props} />;
}

function PopoverTrigger(props: React.ComponentProps<typeof PopoverPrimitive.Trigger>) {
  return <PopoverPrimitive.Trigger data-slot="popover-trigger" {...props} />;
}

function PopoverContent({
  className,
  align = "center",
  sideOffset = 4,
  ...props
}: React.ComponentProps<typeof PopoverPrimitive.Content>) {
  return (
    <PopoverPrimitive.Portal>
      <PopoverPrimitive.Content
        data-slot="popover-content"
        align={align}
        sideOffset={sideOffset}
        className={cn(
          "z-50 w-72 rounded-md border bg-popover p-4 text-popover-foreground shadow-md outline-none",
          className,
        )}
        {...props}
      />
    </PopoverPrimitive.Portal>
  );
}

export { Popover, PopoverTrigger, PopoverContent };
```

- [ ] **Step 2: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: no errors referencing `popover.tsx`. (Per the unified package's API the export is `Popover`. If `tsc` reports the named export differently, read the error and adjust the import to match — same situation as `Accordion` in Phase 1.)

- [ ] **Step 3: Commit**

```bash
git add components/ui/popover.tsx
git commit -m "feat(ui): add Popover primitive (radix-nova)"
```

---

### Task 5: Unified renderer — `ObjectVisual` widget cases + `KioskPreview` over objects

**Files:**
- Modify: `components/device-preview/kiosk-preview.tsx`

Lift the bespoke per-screen JSX into per-widget renderers so both preview and stage render `screen.objects.map(ObjectVisual)`. This is a **refactor-by-extraction**: the visual markup already exists in the `*Screen` functions; move each piece into a widget renderer keyed by object type.

- [ ] **Step 1: Change the `ObjectVisual` signature + add widget cases**

In `kiosk-preview.tsx`, update `ObjectVisual` (currently lines 221–242) to take `config: KioskConfig` instead of `layout: KioskLayout`, and to handle the new types:

```tsx
import {
  type KioskConfig,
  type KioskObject,
  type KioskScreen,
} from "@/lib/kiosk-layout";
import { resolveIconComponent } from "@/lib/kiosk-icons";

export function ObjectVisual({
  object,
  brand,
  config,
}: {
  object: KioskObject;
  brand: KioskBrand;
  config: KioskConfig;
}) {
  switch (object.type) {
    case "text":
      return <TextObject object={object} />;
    case "logo":
      return <LogoObject object={object} brand={brand} />;
    case "clock":
      return <ClockObject object={object} timezone={config.clockTimezone} clock24h={config.clock24h} />;
    case "wifi":
      return <WifiObject object={object} level={config.wifiLevel} />;
    case "icon":
      return <IconObject object={object} brand={brand} />;
    case "qr":
      return <QrObject object={object} />;
    case "spinner":
      return <SpinnerObject object={object} />;
    case "countdown":
      return <CountdownObject object={object} brand={brand} />;
    case "pairingCode":
      return <PairingCodeObject object={object} brand={brand} />;
    case "steps":
      return <StepsObject object={object} />;
    default:
      return null;
  }
}
```

> `ClockObject` currently takes `layout`; change it to `{ object, timezone, clock24h }` and update its body to read those props. (Its internal formatting logic is unchanged.)

- [ ] **Step 2: Add the new widget renderers by lifting existing JSX**

At the bottom of `kiosk-preview.tsx`, add one renderer per new widget. **Lift the inner JSX** from the matching `*Screen` function (line refs below), wrapping it to fill its object box (`size-full` inside the absolutely-positioned wrapper the canvas provides). Use `cq()` for any px font/spacing so it scales. Reference sources:

- `IconObject` — render `resolveIconComponent(object.icon?.preset)` (preset) or an `<img>` of `object.icon?.url` (upload, already presigned by `getTenantBranding`). Apply tint via the existing accent/muted/warn CSS vars and an optional circular background. This replaces the **hard-coded check SVG at lines 421–423**:

```tsx
function IconObject({ object, brand }: { object: KioskObject; brand: KioskBrand }) {
  const ic = object.icon ?? { source: "preset" as const };
  const tintVar =
    ic.tint === "muted" ? "var(--k-muted)" :
    ic.tint === "warn" ? "#E5484D" :
    ic.tint === "none" ? "var(--k-fg)" :
    "var(--k-accent)";
  const Inner = () =>
    ic.source === "upload" && ic.url ? (
      // eslint-disable-next-line @next/next/no-img-element
      <img src={ic.url} alt="" className="size-full object-contain" />
    ) : (
      (() => {
        const Glyph = resolveIconComponent(ic.preset);
        return <Glyph className="size-full" style={{ color: ic.circle ? "var(--k-bg)" : tintVar }} strokeWidth={2.5} />;
      })()
    );
  if (ic.circle) {
    return (
      <div className="flex size-full items-center justify-center rounded-full p-[18%]" style={{ background: tintVar }}>
        <Inner />
      </div>
    );
  }
  return <div className="flex size-full items-center justify-center" style={{ color: tintVar }}><Inner /></div>;
}
```

- `QrObject` — lift the QR block from `ReceiptScreen` (the `<img>`/QR element around line 387 / `e287` in the live render). Render it at `size-full`.
- `SpinnerObject` — lift the spinner element from `ProcessingScreen` (318–340).
- `CountdownObject` — lift the "Code expires 0:48" countdown block from `ReceiptScreen` (the `e522–e524` block).
- `PairingCodeObject` — lift the pairing-code card from `SetupScreen` (the `e591–e594` block); reads `brand.pairingCode`.
- `StepsObject` — lift the numbered steps list from `SetupScreen` (`e580–e589`).

> Keep each renderer **fixed-internal** (the look is fixed, like clock/wifi today) but filling its object box so it can be moved/resized/hidden. Do not invent new visuals — port the existing markup.

- [ ] **Step 3: Rewrite `KioskPreview` to render objects**

Change `KioskPreview` (lines 84–123) to take a `config` + `screen` and render that screen's objects, replacing the per-screen `*Screen` dispatch:

```tsx
export function KioskPreview({
  brand,
  config,
  screen,
  className,
}: {
  brand: KioskBrand;
  config: KioskConfig;
  screen: KioskScreen;
  className?: string;
}) {
  const objects = [...config.screens[screen].objects]
    .filter((o) => o.visible)
    .sort((a, b) => a.z - b.z);
  return (
    <div className={cn("relative aspect-square w-full overflow-hidden", className)} style={kioskRootStyle(brand)}>
      {objects.map((o) => (
        <div
          key={o.id}
          className="absolute"
          style={{ left: `${o.x * 100}%`, top: `${o.y * 100}%`, width: `${o.w * 100}%`, height: `${o.h * 100}%`, zIndex: o.z }}
        >
          <ObjectVisual object={o} brand={brand} config={config} />
        </div>
      ))}
    </div>
  );
}
```

> Delete the now-unused `IdleScreen`/`ProcessingScreen`/`ReceiptScreen`/`SentScreen`/`ErrorScreen`/`PausedScreen`/`SetupScreen` functions once their JSX has been ported into widget renderers. Keep `TextObject`/`LogoObject`/`ClockObject`/`WifiObject`.

- [ ] **Step 4: Verify build + visual smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: clean build (callers of `KioskPreview`/`ObjectVisual` are updated in Tasks 8–11; if build fails only on those call sites, that's expected until then — so verify this task by temporarily updating `branding-editor.tsx`'s `KioskPreview` usage to pass a migrated `config`, OR defer the full build green to Task 11 and here just run `npx tsc --noEmit` on `kiosk-preview.tsx` in isolation). Then `npm run dev` and eyeball each screen via the preview carousel — every screen should look **identical to before** at default seeds.

> Because callers change in later tasks, it is acceptable for `npm run build` to fail on `branding-editor.tsx`/`data.ts` call sites here. Gate this task on: `kiosk-preview.tsx` itself type-checks and the screens render unchanged. Full-tree green returns in Task 11.

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/kiosk-preview.tsx
git commit -m "feat(branding): unified object renderer + per-widget renderers for all screens"
```

---

### Task 6: Generalize `useKioskEditor` to the active screen

**Files:**
- Modify: `components/device-preview/kiosk-editor/use-kiosk-editor.ts`

The hook currently edits a single `KioskLayout`. Make it edit `config.screens[screen].objects` and write shared clock/wifi back to the top-level config.

- [ ] **Step 1: Change the hook input + interface**

Update the `useKioskEditor` signature and the `KioskEditor` interface:

```ts
export function useKioskEditor({
  config,
  screen,
  onChange,
  disabled = false,
}: {
  config: KioskConfig;
  screen: KioskScreen;
  onChange: (c: KioskConfig) => void;
  disabled?: boolean;
}): KioskEditor
```

In the `KioskEditor` interface: replace `layout: KioskLayout; onChange: (l: KioskLayout) => void;` with `config: KioskConfig; screen: KioskScreen; onChange: (c: KioskConfig) => void;`, and add `addIcon: () => void;` next to `addText`. Keep every other method/signature identical (`patch`, `startMove`, `startResize`, pointer handlers, `removeObject`, `bringToFront`, `resetLayout`, `endInteraction`, `isDragging`, `selected`, `selBox`, `guides`, `ordered`, `atCustomCap`).

- [ ] **Step 2: Operate on the active screen**

Inside the hook, derive the active screen's objects and write changes back into the config:

```ts
  const objects = config.screens[screen].objects;

  // Replace the whole active screen's object list, preserving other screens + shared config.
  const setObjects = (next: KioskObject[]) =>
    onChange({ ...config, screens: { ...config.screens, [screen]: { objects: next } } });
```

Rewrite the internal handlers that previously called `onChange({ ...layout, objects })` to call `setObjects(next)`. For shared clock/wifi edits (timezone/clock24h/wifiLevel — currently patched on the layout), write to the top level instead:

```ts
  const setShared = (p: Partial<Pick<KioskConfig, "clockTimezone" | "clock24h" | "wifiLevel">>) =>
    onChange({ ...config, ...p });
```

> `kiosk-controls.tsx` clock/wifi controls (Task 7) call `setShared` via new editor methods OR via `patch` special-casing — expose `setShared` on the interface as `setShared: (p: ...) => void;` and add it to the returned object. Update the `KioskEditor` interface accordingly.

`addText` uses `createTextObject`; add `addIcon` using `createIconObject`:

```ts
  const addIcon = () => {
    if (disabled || atCustomCap) return;
    const z = objects.reduce((m, o) => Math.max(m, o.z), 0) + 1;
    setObjects([...objects, createIconObject(z)]);
  };
```

`atCustomCap` now counts **text + icon** objects on the active screen against `MAX_CUSTOM`:

```ts
  const addableCount = objects.filter((o) => o.type === "text" || o.type === "icon").length;
  const atCustomCap = addableCount >= MAX_CUSTOM;
```

`resetLayout` resets **the active screen** to its seed:

```ts
  const resetLayout = () => { if (!disabled) setObjects(seededScreen(screen).objects); };
```

Import `createIconObject`, `seededScreen`, `MAX_CUSTOM`, and the v3 types from `@/lib/kiosk-layout`.

- [ ] **Step 3: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: errors only at call sites in `kiosk-stage.tsx`/`kiosk-controls.tsx`/`branding-editor.tsx` (fixed in Tasks 7, 11). The hook file itself has no internal type errors.

- [ ] **Step 4: Commit**

```bash
git add components/device-preview/kiosk-editor/use-kiosk-editor.ts
git commit -m "feat(branding): editor operates on the active screen + addIcon"
```

---

### Task 7: Type-aware Properties + icon picker (`kiosk-controls.tsx`, `kiosk-icon-picker.tsx`)

**Files:**
- Create: `components/device-preview/kiosk-icon-picker.tsx`
- Modify: `components/device-preview/kiosk-editor/kiosk-controls.tsx`

- [ ] **Step 1: Create the icon picker**

Create `components/device-preview/kiosk-icon-picker.tsx`:

```tsx
"use client";

import * as React from "react";
import { Popover, PopoverTrigger, PopoverContent } from "@/components/ui/popover";
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs";
import { Button } from "@/components/ui/button";
import { ICON_PRESETS, type KioskIcon, type IconPreset, type IconTint } from "@/lib/kiosk-layout";
import { ICON_COMPONENTS } from "@/lib/kiosk-icons";
import { cn } from "@/lib/utils";

const TINTS: { value: IconTint; label: string }[] = [
  { value: "accent", label: "Accent" },
  { value: "muted", label: "Muted" },
  { value: "warn", label: "Warn" },
  { value: "none", label: "None" },
];

export function KioskIconPicker({
  icon,
  disabled,
  onChange,
  onUpload,
}: {
  icon: KioskIcon;
  disabled?: boolean;
  onChange: (next: KioskIcon) => void;
  onUpload: (file: File) => void;
}) {
  const fileRef = React.useRef<HTMLInputElement>(null);
  const Active = ICON_COMPONENTS[(icon.preset ?? "check") as IconPreset];

  return (
    <Popover>
      <PopoverTrigger asChild>
        <Button type="button" variant="outline" size="sm" disabled={disabled} className="gap-2">
          {icon.source === "upload" ? <span className="text-xs">Custom image</span> : <Active className="size-4" />}
          Choose icon
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-72 space-y-3">
        <Tabs defaultValue="library">
          <TabsList className="w-full">
            <TabsTrigger value="library" className="flex-1">Library</TabsTrigger>
            <TabsTrigger value="upload" className="flex-1">Upload</TabsTrigger>
          </TabsList>
          <TabsContent value="library">
            <div className="grid grid-cols-6 gap-1.5">
              {ICON_PRESETS.map((name) => {
                const Glyph = ICON_COMPONENTS[name];
                const active = icon.source === "preset" && icon.preset === name;
                return (
                  <button
                    key={name}
                    type="button"
                    aria-label={name}
                    onClick={() => onChange({ ...icon, source: "preset", preset: name })}
                    className={cn(
                      "flex aspect-square items-center justify-center rounded-md border text-muted-foreground hover:bg-accent",
                      active && "border-foreground text-foreground ring-1 ring-foreground",
                    )}
                  >
                    <Glyph className="size-4" />
                  </button>
                );
              })}
            </div>
          </TabsContent>
          <TabsContent value="upload">
            <input
              ref={fileRef}
              type="file"
              accept="image/svg+xml,image/png"
              className="hidden"
              onChange={(e) => {
                const f = e.target.files?.[0];
                if (f) onUpload(f);
              }}
            />
            <Button type="button" variant="outline" size="sm" className="w-full" onClick={() => fileRef.current?.click()} disabled={disabled}>
              Upload SVG or PNG (≤2 MB)
            </Button>
          </TabsContent>
        </Tabs>

        <div className="space-y-1.5">
          <p className="text-xs font-medium text-muted-foreground">Tint</p>
          <div className="flex gap-1">
            {TINTS.map((t) => (
              <button
                key={t.value}
                type="button"
                onClick={() => onChange({ ...icon, tint: t.value })}
                className={cn("flex-1 rounded-md border px-1 py-1 text-xs hover:bg-accent", (icon.tint ?? "accent") === t.value && "border-foreground ring-1 ring-foreground")}
              >
                {t.label}
              </button>
            ))}
          </div>
        </div>

        <label className="flex items-center justify-between text-xs">
          <span className="font-medium text-muted-foreground">Circle background</span>
          <input type="checkbox" checked={!!icon.circle} onChange={(e) => onChange({ ...icon, circle: e.target.checked })} />
        </label>
      </PopoverContent>
    </Popover>
  );
}
```

- [ ] **Step 2: Wire the picker + `Add icon` into `kiosk-controls.tsx`**

In `kiosk-controls.tsx`:
- Add an **Add icon** button next to **Add text** (calls `editor.addIcon()`, disabled when `editor.atCustomCap`).
- In the `Properties` component, add an **icon branch** (when `object.type === "icon"`): render `<KioskIconPicker icon={object.icon ?? { source: "preset", preset: "check", tint: "accent" }} disabled={editor.disabled} onChange={(next) => editor.patch(object.id, { icon: next })} onUpload={(file) => onIconUpload(object.id, file)} />`.
- The clock/wifi/timezone controls now call the new shared setters from Task 6 (`editor.setShared({ clockTimezone })`, etc.) instead of patching a layout-level field.
- `onIconUpload(objectId, file)` is a new prop on `KioskControls` (the upload must reach `branding-editor.tsx`'s file map — Task 11). Add it to the component's props: `{ editor, onIconUpload }: { editor: KioskEditor; onIconUpload: (objectId: string, file: File) => void }`.

> Validate the uploaded file in `branding-editor.tsx` (Task 11): image type + ≤2 MB, same as the logo. The picker just forwards the `File`.

- [ ] **Step 3: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: errors only at the `KioskControls` call site in `branding-editor.tsx` (now needs `onIconUpload`) — fixed in Task 11. `kiosk-controls.tsx` + `kiosk-icon-picker.tsx` themselves are clean.

- [ ] **Step 4: Commit**

```bash
git add components/device-preview/kiosk-icon-picker.tsx components/device-preview/kiosk-editor/kiosk-controls.tsx
git commit -m "feat(branding): icon picker (library + upload) and Add icon control"
```

---

### Task 8: `kiosk-stage.tsx` renders the active screen's objects

**Files:**
- Modify: `components/device-preview/kiosk-editor/kiosk-stage.tsx`

The stage already maps `editor.ordered` to draggable wrappers + `ObjectVisual`. Update it for the new `ObjectVisual` signature and ensure double-click inline edit stays text-only.

- [ ] **Step 1: Update the ObjectVisual call + config source**

In `kiosk-stage.tsx`, pass `config` to `ObjectVisual`. The editor exposes the config, so render:

```tsx
<ObjectVisual object={o} brand={brand} config={editor.config} />
```

The inline text editor (`InlineTextEditor`) and the `onDoubleClick` guard already check `o.type === "text"` — leave them as-is. Resize/drag/selection overlays are generic over objects and need no change.

- [ ] **Step 2: Verify it type-checks + build**

Run: `npx tsc --noEmit`
Expected: errors only at the `branding-editor.tsx` call site (Task 11). `kiosk-stage.tsx` itself is clean.

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/kiosk-editor/kiosk-stage.tsx
git commit -m "feat(branding): stage renders active screen via unified ObjectVisual"
```

---

### Task 9: Schema — add `tenant_settings.kioskScreens` + migration

**Files:**
- Modify: `lib/db/schema.ts`
- Create: `lib/db/migrations/<generated>.sql`

Keep `kioskLayout` for one release (rollback safety); add the v3 column.

- [ ] **Step 1: Add the column**

In `lib/db/schema.ts` `tenantSettings`, after `kioskLayout: jsonb("kiosk_layout"),` add:

```ts
  kioskScreens: jsonb("kiosk_screens"),
```

- [ ] **Step 2: Generate the migration**

Run: `npm run db:generate`
Expected: a new SQL file under `lib/db/migrations/` with `ALTER TABLE "tenant_settings" ADD COLUMN "kiosk_screens" jsonb;`.

- [ ] **Step 3: Verify the migration applies**

Run: `npm run db:migrate`
Expected: applies cleanly to Neon (no error). The column is nullable, so existing rows are unaffected; `getTenantBranding` (Task 10) backfills-on-read via `normalizeKioskConfig` (which migrates the old `kioskLayout` when `kioskScreens` is null).

> No data backfill SQL is required: read-time normalization migrates v2→v3, and the first `saveBranding` persists the v3 config. This mirrors how `kioskLayout` itself rolled out.

- [ ] **Step 4: Commit**

```bash
git add lib/db/schema.ts lib/db/migrations
git commit -m "feat(branding): add tenant_settings.kiosk_screens (v3 config) column"
```

---

### Task 10: Persistence — `getTenantBranding` returns `KioskConfig`; `saveBranding` writes it

**Files:**
- Modify: `lib/storage.ts`, `lib/data.ts`, `app/(tenant)/tenant/branding/actions.ts`

- [ ] **Step 1: Add an icon storage key helper**

In `lib/storage.ts`, after `logoStorageKey` (line ~98), add:

```ts
export function iconStorageKey(organizationId: string, assetId: string): string {
  return `branding/${organizationId}/icons/${assetId}`;
}
```

- [ ] **Step 2: `getTenantBranding` → normalized config + presigned icons**

In `lib/data.ts`, update the `TenantBranding` interface: replace `kioskLayout: KioskLayout;` with `kioskConfig: KioskConfig;` (import `KioskConfig`, `normalizeKioskConfig`, `KIOSK_SCREENS` from `@/lib/kiosk-layout`). In `getTenantBranding`:

```ts
  // Prefer v3 kioskScreens; fall back to migrating the legacy kioskLayout.
  const config = normalizeKioskConfig(s?.kioskScreens ?? s?.kioskLayout);

  // Presign every uploaded icon key across all screens (collect → presign → map back).
  const iconKeys = new Set<string>();
  for (const screen of KIOSK_SCREENS) {
    for (const o of config.screens[screen].objects) {
      if (o.type === "icon" && o.icon?.source === "upload" && o.icon.url) iconKeys.add(o.icon.url);
    }
  }
  const signed = new Map<string, string>();
  await Promise.all([...iconKeys].map(async (k) => signed.set(k, await presignedGetUrl(k))));
  for (const screen of KIOSK_SCREENS) {
    for (const o of config.screens[screen].objects) {
      if (o.type === "icon" && o.icon?.source === "upload" && o.icon.url) {
        o.icon = { ...o.icon, url: signed.get(o.icon.url) ?? o.icon.url };
      }
    }
  }
```

Return `kioskConfig: config` instead of `kioskLayout`.

> The presigned URLs are display-only. On save, the editor must send back the **stored keys**, not presigned URLs — handle that in Step 3 by keeping a key map (or by treating any `url` that isn't a freshly-uploaded file as the existing key; since presigned URLs differ from keys, store the original key alongside — simplest: in `getTenantBranding` also return a `iconKeyByUrl` map, OR have `saveBranding` re-normalize and re-resolve. **Chosen approach:** `saveBranding` receives the full config where upload icons carry either a `url` that is an existing R2 key (unchanged) or a sentinel `pending:<objectId>` for newly-uploaded files; see Step 3.)

- [ ] **Step 3: `saveBranding` accepts the v3 config + icon files**

In `app/(tenant)/tenant/branding/actions.ts`:
- Read `formData.get("kioskScreens")` (JSON string) instead of/alongside `kioskLayout`. Parse and `normalizeKioskConfig`.
- For each newly-uploaded icon, the client sends a file under key `icon:<objectId>` and sets that object's `icon.url = "pending:<objectId>"`. For each `pending:` object: read the file, validate (image, ≤2 MB), `putObject(iconStorageKey(orgId, nanoid()), bytes, type)`, then rewrite that object's `icon.url` to the new key.
- Persist the rewritten, normalized config to `kioskScreens` via the existing `onConflictDoUpdate` upsert (add `kioskScreens` to both the insert values and the update set, guarded by `!== undefined` like `kioskLayout`).
- Best-effort cleanup: diff the previous config's upload keys vs the new set; `deleteObject` orphaned keys (mirror the logo cleanup at lines 135–142, generalized to a set of keys).
- Auth gate (owner/admin) and audit log unchanged.

> Keep writing `kioskLayout` too (the idle screen, as v2) for one release so a rollback still renders. Derive it from `config.screens.idle`: `{ version: 2, clockTimezone, clock24h, wifiLevel, objects: config.screens.idle.objects }`.

- [ ] **Step 4: Verify type-check + build (expects Task 11 for full green)**

Run: `npx tsc --noEmit`
Expected: the only remaining error is `branding-editor.tsx` still reading `kioskLayout` (fixed in Task 11). `data.ts`/`actions.ts`/`storage.ts` are clean.

- [ ] **Step 5: Commit**

```bash
git add lib/storage.ts lib/data.ts "app/(tenant)/tenant/branding/actions.ts"
git commit -m "feat(branding): persist + presign v3 kiosk config with icon uploads"
```

---

### Task 11: Wire `branding-editor.tsx` to the v3 config (every screen editable)

**Files:**
- Modify: `components/branding-editor.tsx`

This closes the loop: drive the whole editor from `KioskConfig` + the active `screen`, make **every** screen editable (remove the Phase 1 "Per-screen editing arrives in the next update" placeholder), collect icon uploads, and submit the config.

- [ ] **Step 1: Switch props + state from layout to config**

Change the `BrandingEditor` prop `initialLayout: KioskLayout` → `initialConfig: KioskConfig`. Replace `const [layout, setLayout] = React.useState(initialLayout)` with `const [config, setConfig] = React.useState(initialConfig)`. The page that renders `<BrandingEditor>` (`app/(tenant)/tenant/branding/page.tsx`) passes `initialConfig={branding.kioskConfig}` (from the updated `getTenantBranding`).

- [ ] **Step 2: Editor + dirty flag + icon-file state**

```tsx
  const editor = useKioskEditor({ config, screen, onChange: setConfig, disabled: !canEdit });

  // Newly-uploaded icon files, keyed by object id, flushed on save.
  const [iconFiles, setIconFiles] = React.useState<Record<string, File>>({});
  const onIconUpload = (objectId: string, file: File) => {
    if (!file.type.startsWith("image/")) { toast.error("Icon must be an image."); return; }
    if (file.size > 2 * 1024 * 1024) { toast.error("Icon must be under 2 MB."); return; }
    setIconFiles((m) => ({ ...m, [objectId]: file }));
    const current = editor.config.screens[screen].objects.find((o) => o.id === objectId)?.icon ?? { source: "preset" as const };
    editor.patch(objectId, { icon: { ...current, source: "upload", url: `pending:${objectId}` } });
  };
```

Update the `dirty` check to compare `JSON.stringify(config) !== JSON.stringify(initialConfig)` (replacing the `layout` comparison) and include `Object.keys(iconFiles).length > 0`.

- [ ] **Step 3: Make the accordion "{screen} content" editable for all screens**

Replace the Phase 1 placeholder branch (the `screen === "idle" ? <KioskControls .../> : <placeholder/>`) with `<KioskControls editor={editor} onIconUpload={onIconUpload} />` unconditionally — the editor is now screen-aware, so it works for every screen.

- [ ] **Step 4: Render the editable stage for every screen**

In the preview carousel `renderSlide`, replace the idle-only `KioskStage` with `KioskStage` for **the active screen** and `KioskPreview` (read-only) for the others:

```tsx
renderSlide={(i) =>
  SCREENS[i].value === screen ? (
    <KioskStage editor={editor} brand={kioskBrand} />
  ) : (
    <KioskPreview brand={kioskBrand} config={config} screen={SCREENS[i].value} />
  )
}
```

(`KioskStage` already reads `editor.config`/`editor.screen`; `KioskPreview` now takes `config`.)

- [ ] **Step 5: Submit the config + icon files**

In `save()`, replace `fd.set("kioskLayout", JSON.stringify(layout))` with:

```tsx
    fd.set("kioskScreens", JSON.stringify(config));
    for (const [objectId, file] of Object.entries(iconFiles)) fd.set(`icon:${objectId}`, file);
```

On success, `setIconFiles({})`.

- [ ] **Step 6: Verify full build + Playwright smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: **clean, whole-tree green** (this is the task that closes all the deferred call-site errors). Remove any now-unused imports (`KioskLayout`, etc.).

Then `npm run dev`, sign in `dana@roastwell.co` / `123456`, open **/tenant/branding** and verify:
- Each of the 7 screens renders identically to before at defaults (carousel through all).
- Switch to **Sent ✓**, select the check icon, open the picker → change preset (e.g. `party-popper`) and toggle circle/tint → preview updates live; the save bar shows "Unsaved changes".
- Upload a custom PNG/SVG icon → it renders in the preview.
- Drag/resize objects on a non-idle screen; double-click a text label to edit it inline.
- **Save** → toast success → `router.refresh()` → reload persists the change (icon preset + position survive).
- View-only role (a `member`) disables all controls.

- [ ] **Step 7: Commit**

```bash
git add components/branding-editor.tsx "app/(tenant)/tenant/branding/page.tsx"
git commit -m "feat(branding): full per-screen editing + editable Success icon"
```

---

## Self-Review Notes

- **Spec coverage (Phase 2):**
  - §2.1 data model → Task 1 (types) + Task 2 (config).
  - §2.2 seeded defaults → Task 1 (`seededScreen`, all 7 screens; idle reuses v2 default; sent/error icons asserted by test).
  - §2.3 unified renderer → Task 5 (`ObjectVisual` widget cases + `KioskPreview` over objects; bespoke `*Screen` JSX lifted/removed).
  - §2.4 icon picker → Task 4 (Popover primitive) + Task 7 (picker with Library/Upload tabs, tint, circle).
  - §2.5 editor generalization → Task 6 (`useKioskEditor` per-screen, `addIcon`, per-screen reset, shared clock/wifi) + Task 8 (stage) + Task 7 (controls).
  - §2.6 persistence & migration → Task 9 (schema/migration) + Task 10 (`getTenantBranding` presign-all + `saveBranding` config + icon uploads) + Task 2 (`normalizeKioskConfig`).
- **No-visual-change guarantee:** seeds reproduce the current templates; read-time `normalizeKioskConfig` migrates legacy `kioskLayout`→idle and seeds the rest; verified in Task 11 Step 6 by carouseling all screens at defaults.
- **Type consistency:** `KioskConfig`/`ScreenLayout`/`KioskIcon`/`IconPreset`/`IconTint` defined in Task 1 are used unchanged in Tasks 2, 5, 6, 7, 10, 11. `ObjectVisual({object, brand, config})` (Task 5) is called identically in `kiosk-stage.tsx` (Task 8) and `KioskPreview` (Task 5). `useKioskEditor({config, screen, onChange, disabled})` (Task 6) is called with exactly those props in `branding-editor.tsx` (Task 11). `addIcon`/`setShared`/`atCustomCap`(text+icon) added to the `KioskEditor` interface in Task 6 are consumed in Task 7. `iconStorageKey` (Task 10 Step 1) is used in Task 10 Step 3. `kioskScreens` column (Task 9) is read in Task 10 Step 2 and written in Task 10 Step 3.
- **Deferred-build caveat is explicit:** Tasks 5–10 each note that whole-tree `npm run build` may fail only at not-yet-updated call sites; full green is gated on Task 11. Each of those tasks is still independently verified (`tsc` on the changed file + unit tests where applicable). This is intentional given the renderer/editor signature change ripples through several call sites; the alternative (one giant task) would violate bite-sized granularity.
- **Rollback safety:** `kioskLayout` (v2) is retained and still written from `screens.idle` for one release, so reverting the deploy keeps idle rendering.
- **Open verification risk flagged in-plan:** the `seededScreen` pixel positions in Task 1 are approximations of the current JSX; Task 5 Step 4 and Task 11 Step 6 gate on a visual diff against the live render. If a screen looks off, adjust the seed boxes — the test only asserts structural validity, not exact pixels, by design.
```
