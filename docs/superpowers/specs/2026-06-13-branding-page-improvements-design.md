# Branding Page Improvements — Full-Screen Editor & Compact Studio Design

**Date:** 2026-06-13
**Status:** Approved design, pending implementation plan
**Goal:** Make the Branding page fully usable and internally consistent: every kiosk screen becomes editable (not just idle), the Success Icon and all labels are editable, the left settings panel becomes compact, and the live preview gains a zoom slider and swipe navigation — all without changing how the kiosk screens look out of the box.

## Summary

The Branding page today is half-finished in a way users feel: the **idle** screen
is a rich drag-and-drop object editor, while the other six screens (`processing`,
`qr`, `sent`, `error`, `paused`, `setup`) are hard-coded JSX templates you can only
look at. That split is the root of every complaint:

1. **"I can add labels but can't edit existing ones."** Text *is* editable, but only
   after selecting an object via the Objects list (which then reveals a Properties
   panel). New labels work because `addText()` auto-selects; existing labels have no
   obvious affordance and no inline edit.
2. **"I can't change the Success Icon."** The Sent ✓ checkmark is a fixed SVG
   (`kiosk-preview.tsx:421`); nothing on the non-idle screens is editable.
3. **Left panel too large.** Three always-expanded stacked cards (Brand / Idle layout
   / Security) consume a tall column.
4. **Preview too large** and **no way to switch screens by swiping.**

The unifying fix is to **collapse the two rendering paths into one**: every screen
becomes a list of positionable objects rendered by a single engine, and each screen's
default layout is **seeded from its current template** so the out-of-box look is
pixel-identical. On top of that we add an `icon` object type (curated + uploadable),
a compact accordion panel, a preview zoom slider, and a swipeable screen carousel.

**Approved scope (must-haves):**
- **Full editor on every screen** — all 7 screens use the object model; add / edit /
  remove / drag / resize / show-hide on each. Defaults seeded from today's templates
  so styling is unchanged.
- **Editable Success Icon** via a new **`icon`** object type, sourced from a **curated
  lucide set _and_ custom upload** ("Both"), tinted by brand color.
- **Inline label editing** — double-click a text object on the canvas to edit in place.
- **Compact accordion** left panel: Brand · This screen · Security.
- **Preview zoom slider** (`%`) and **swipe/carousel** screen navigation.

**Out of scope (YAGNI):** per-object color/font-family pickers beyond what exists
(size + align); animation editing; reordering screens; multi-icon libraries beyond the
curated allowlist; templating/sharing layouts across tenants.

## Current state (what exists today)

- `lib/kiosk-layout.ts` — `KioskObject { id, type: "text"|"logo"|"clock"|"wifi", x,y,w,h
  (fractions 0..1), visible, z, text?, fontSize?, align? }`, `KioskLayout { version: 2,
  clockTimezone, clock24h, wifiLevel, objects[] }`, `normalizeKioskLayout`,
  `createTextObject`, `objectLabel`, `defaultLayout`, `MAX_CUSTOM=20`, `MAX_TEXT_LEN=80`,
  `FONT_MIN/MAX`. (+ test file.)
- `lib/kiosk-geometry.ts` — pure box move/resize/snap over `{x,y,w,h}` fractions.
- `components/device-preview/kiosk-editor/use-kiosk-editor.ts` — `useKioskEditor` hook:
  selection, drag, snap guides, `addText`/`removeObject`/`patch`/`bringToFront`/`resetLayout`.
- `components/device-preview/kiosk-editor/kiosk-stage.tsx` — editable canvas + selection
  overlay/handles. Renders the **idle** screen only.
- `components/device-preview/kiosk-editor/kiosk-controls.tsx` — Objects list +
  type-aware `Properties` (text/clock/wifi) + reset.
- `components/device-preview/kiosk-preview.tsx` — `KioskPreview` renders all 7 screens;
  idle from objects (`ObjectVisual`), the **other six are hard-coded JSX** (Sent checkmark
  SVG, error icon, spinner, QR block, countdown bar, pairing code, steps). Exports
  `ObjectVisual`, `KioskScreen`, `KioskBrand`, `cq`, `kioskRootStyle`.
- `components/branding-editor.tsx` — two-pane studio: left = 3 cards; right = sticky Live
  preview card with a screen `Select`; bottom = sticky save bar. Holds all brand/theme/
  logo/pin/screen state.
- `app/(tenant)/tenant/branding/{page.tsx,actions.ts}` — load via `getTenantBranding`,
  persist via `saveBranding`. `tenantSettings.kioskLayout` jsonb holds the single layout.
- `lib/data.ts` `getTenantBranding` — resolves theme tokens, normalizes layout, presigns
  the logo URL.

## Design

The work splits into **two independently shippable phases** so the visible polish lands
fast and low-risk, and the deep data-model change is isolated.

---

## Phase 1 — UX shell (no data-model change)

Pure front-end. Touches `branding-editor.tsx`, `kiosk-stage.tsx`, `kiosk-controls.tsx`,
and a small new preview-frame wrapper. No schema/action/data changes.

### 1.1 Compact accordion panel

Replace the three stacked `Section` cards with a single **accordion** (shadcn
`Accordion`, `type="single"`, collapsible) with three items:

- **Brand** (global) — logo, logo text, accent color + presets, advanced theme tokens.
- **This screen** (contextual) — retitles to the active screen ("Idle layout", "Sent ✓
  content", …). Holds the `KioskControls` for that screen. In Phase 1 this still only has
  real controls on idle; other screens show the existing "switch to idle to edit" hint.
  Phase 2 makes it live for every screen.
- **Security** (global) — staff PIN.

Default-open the **This screen** item (the primary work surface). Section icons/titles
reuse the current `Palette`/`LayoutGrid`/`ShieldCheck` treatment so it stays visually
consistent with the rest of the app.

### 1.2 Preview zoom slider

Add a zoom control to the Live-preview card header: a shadcn `Slider` (or compact
`−  82%  +`) bound to a `scale` state, **default 80%**, range **50–125%**, step 5%. The
preview canvas wrapper applies `transform: scale(scale)` with `transform-origin: top
center` and reserves layout height accordingly so the card doesn't jump. Zoom is
**visual only** — it never touches the saved 720×720 geometry. Pointer math in
`useKioskEditor` already derives fractions from the canvas `getBoundingClientRect()`, so
drag stays accurate at any zoom (the rect reflects the scaled size).

### 1.3 Swipe / carousel screen navigation

Wrap the preview canvas in a carousel:
- Horizontal **swipe/drag** between screens (pointer/touch), **prev/next arrows**, and
  **dot indicators** for the 7 screens.
- Keep the existing **dropdown** as a jump-to control.
- **Gesture guard:** swipe is disabled while an object drag is in progress
  (`useKioskEditor` exposes interaction state) so dragging an element never flips screens.
- Implementation: a lightweight controlled carousel over the existing screen list (we can
  use the shadcn/embla `Carousel` already available, or a minimal translate-X track).
  Decision deferred to the plan; behavior is the contract.

### 1.4 Inline label editing (affordance fix)

- **Double-click** a `text` object on the canvas → it becomes an inline `contentEditable`/
  overlay input seeded with the current text; Enter/blur commits via `editor.patch`,
  Escape cancels. This fixes the "can't edit existing labels" complaint directly.
- Selection affordance is also clearer because the **This screen** accordion item
  surfaces the Objects list + Properties for the active screen by default.

---

## Phase 2 — Per-screen object model + icon system

This delivers "full editor everywhere" and the editable Success Icon. It generalizes the
single-screen layout to all seven and unifies the renderer.

### 2.1 Data model (`lib/kiosk-layout.ts`)

```ts
type KioskScreen =
  | "idle" | "processing" | "qr" | "sent" | "error" | "paused" | "setup";

type KioskObjectType =
  | "text" | "logo" | "clock" | "wifi"        // existing
  | "icon"                                     // new — curated glyph or uploaded image
  | "qr" | "spinner" | "countdown"            // new — positionable fixed-render widgets
  | "pairingCode" | "steps";                  //       (move/resize/show-hide; look fixed)

interface KioskIcon {
  source: "preset" | "upload";
  preset?: string;            // lucide name from a curated allowlist (source=preset)
  url?: string;               // R2 object key (source=upload); presigned on read
  tint?: "accent" | "muted" | "warn" | "none"; // default "accent"
  circle?: boolean;           // filled circular background (the Sent ✓ look)
}

interface KioskObject {
  id: string;
  type: KioskObjectType;
  x: number; y: number; w: number; h: number; // fractions 0..1 on the 720² canvas
  visible: boolean;
  z: number;
  // text:
  text?: string; fontSize?: number; align?: TextAlign;
  // icon:
  icon?: KioskIcon;
}

interface ScreenLayout { objects: KioskObject[]; }

interface KioskConfig {
  version: 3;
  clockTimezone: string;      // shared (clock widget)
  clock24h: boolean;          // shared
  wifiLevel: number;          // shared (wifi widget)
  screens: Record<KioskScreen, ScreenLayout>;
}
```

Clock/wifi/timezone are promoted to shared top-level config because they're device-wide,
not per-screen. `MAX_CUSTOM` applies **per screen** to `text`+`icon` objects.

### 2.2 Seeded defaults — styling preserved by construction

A `seededScreen(screen, ctx)` builder reproduces each current template as objects at the
same positions/sizes/fonts/colors the JSX uses today (measured against the 720² canvas via
the existing `cq()` reference scale). Examples:
- **sent** → an `icon` object `{ preset: "check", circle: true, tint: "accent" }` at the
  current circle position/size, plus two `text` objects for the title and subtext, plus the
  "Returning to start…" footer text.
- **error** → `icon { preset: "wifi-off", tint: "warn" }`, headline/subtext text, the tan
  "ask for a paper receipt" pill (a styled `text`).
- **qr** → `qr` widget + heading/subtext text + `countdown` widget.
- **processing** → `spinner` widget + caption text.
- **setup** → `pairingCode` widget + `qr` + `steps` widget + heading.
- **paused** → `logo` (dimmed) + text.
- **idle** → unchanged (already object-based; migrated as-is).

Because defaults match the templates, a tenant who never opens the new screens sees no
visual change.

### 2.3 Unified renderer

`ObjectVisual` (already exported) gains cases for `icon`/`qr`/`spinner`/`countdown`/
`pairingCode`/`steps` by **lifting the existing per-screen JSX into per-widget renderers**.
`KioskPreview` and `KioskStage` both render `screen.objects.map(ObjectVisual)`; the only
difference is `KioskStage` adds selection/drag overlays. The bespoke per-screen `*Screen`
functions in `kiosk-preview.tsx` are refactored away into widget renderers + seeded
defaults. The new widgets are **fixed-render** (internal look fixed, like clock/wifi today)
but movable/resizable/hideable.

### 2.4 Icon picker (Properties panel)

When an `icon` object is selected, `kiosk-controls.tsx` `Properties` shows an **icon
picker popover** with two tabs:
- **Library** — a curated grid of ~16–24 lucide icons (allowlist constant, e.g. Check,
  CheckCircle, Heart, Star, Gift, Mail, ThumbsUp, Smile, Clock, Bell, AlertTriangle,
  WifiOff, Sparkles, PartyPopper…), rendered tinted by accent.
- **Upload** — drag/drop SVG/PNG (≤2 MB), same validation as the logo, stored in R2.

Plus toggles for **tint** (accent/muted/warn/none) and **circle** background. The lucide
allowlist maps names→components **client-side only** (no server→client function passing,
per the project gotcha).

### 2.5 Editor generalization (`use-kiosk-editor.ts`, `kiosk-stage.tsx`, `kiosk-controls.tsx`)

- `useKioskEditor` operates on **the active screen's** `ScreenLayout` instead of a single
  layout. The hook takes `{ config, screen, onChange, disabled }`, edits
  `config.screens[screen].objects`, and writes shared clock/wifi back to top level.
- `addText` stays; add **`addIcon`** (seeds a default `icon` object). The Objects list,
  show/hide, delete, drag, resize, snap, bring-to-front all work unchanged because they're
  already generic over objects.
- `resetLayout` resets **the active screen** to its seeded default (per-screen reset),
  with the existing "Reset layout to default" affordance.

### 2.6 Persistence & migration

- **Schema:** add `tenantSettings.kioskScreens jsonb`. Keep `kioskLayout` for one release
  for safe rollback; a Drizzle migration backfills `kioskScreens` = `{ version:3, clock*,
  wifiLevel, screens: { idle: <old layout objects>, …seeded others } }`.
- **`normalizeKioskConfig(raw, ctx)`** coerces any stored shape into a valid v3 config:
  fills missing screens from seeds, caps per-screen objects, validates icon source/preset
  against the allowlist, drops unknown presets to a safe default, clamps geometry — mirrors
  today's `normalizeKioskLayout` robustness.
- **`getTenantBranding`** returns the normalized `KioskConfig`; it presigns **all** icon
  upload keys across every screen (collect keys → presign → map back onto objects), the
  same way the logo is presigned today.
- **`saveBranding`** accepts the full config JSON + any newly uploaded icon files
  (multipart, keyed by object id), uploads them to `branding/{orgId}/icons/{nanoid}`,
  rewrites those objects' `icon.url` to the new keys, normalizes, and upserts. Best-effort
  cleanup of orphaned icon keys (mirrors logo cleanup). Owner/admin gate + audit log
  unchanged.

## Components & boundaries

| Unit | Responsibility | Depends on |
|---|---|---|
| `lib/kiosk-layout.ts` | v3 types, seeds, `normalizeKioskConfig`, `createTextObject`/`createIconObject`, allowlist | — |
| `lib/kiosk-icons.ts` (new) | curated lucide allowlist + name→component map (client) | lucide-react |
| `use-kiosk-editor.ts` | per-screen object editing state/handlers | kiosk-layout, kiosk-geometry |
| `kiosk-stage.tsx` | editable canvas for active screen + inline text edit | use-kiosk-editor, ObjectVisual |
| `kiosk-controls.tsx` | Objects list + type-aware Properties incl. icon picker | use-kiosk-editor, kiosk-icons |
| `kiosk-preview.tsx` | unified `ObjectVisual` + widget renderers (no per-screen JSX) | kiosk-layout, kiosk-icons |
| `branding-editor.tsx` | accordion shell, zoom slider, carousel, save/reset | all of the above |
| `actions.ts` / `data.ts` | persist/normalize/presign v3 config + icon uploads | kiosk-layout, storage |

## Testing

- **Pure units (vitest):** `normalizeKioskConfig` (legacy v2→v3 migration, missing screens,
  bad icon preset, geometry clamping, per-screen caps); `seededScreen` produces valid
  objects for all 7 screens; icon allowlist resolution.
- **Migration:** generated SQL applies cleanly; backfill maps an existing tenant's idle
  layout into `screens.idle` and seeds the rest.
- **Manual / Playwright smoke:** each screen renders identically to the current template at
  default; edit a label inline; change the Sent icon (preset + upload); zoom slider; swipe
  between screens; save → refresh persists; view-only role disables all controls.

## Build order

1. **Phase 1** (own plan): accordion panel, zoom slider, swipe carousel, inline label edit.
   Ships visible polish with zero data-model risk.
2. **Phase 2** (own plan): v3 config + migration, unified renderer + widgets, `icon` type +
   picker, per-screen editor generalization, persistence/presigning.

Each phase gets its own implementation plan (writing-plans) and is verified before the next.
