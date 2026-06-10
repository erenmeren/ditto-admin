# Kiosk Screen Editor — Box-Model Rework Design

**Date:** 2026-06-10
**Status:** Approved design, pending implementation plan
**Goal:** Rebuild the kiosk idle-screen editor on a practical SquareLine/LVGL-style **real box model** (objects with pixel width/height) so move/resize feel right, content never distorts, and all text is editable.

## Summary

The current editor sizes objects with `sx/sy` scale multipliers applied as a CSS
`transform: scale(...)`. That model is the root of three reported problems:
1. **Move "jump"/distortion** — `startMove` records no grab-offset; the first
   pointer-move snaps the object's *center* to the cursor.
2. **Resize distortion + fixed-thickness border** — resizing changes a CSS scale
   factor, stretching content (non-uniform on edge drags) while the selection
   ring/handles keep their fixed pixel thickness.
3. **Text not editable** — only custom text boxes have a text field; the built-in
   **Tagline** is hard-coded and **Lane** is derived from the store name.

The fix is a model change: every object becomes a **real box** with a position
and size in pixels; content lays out *inside* the box; font size is its own
property; borders/handles are real CSS on real boxes. Move only changes position;
resize changes the box's `w`/`h`. This is the SquareLine/LVGL mental model and
resolves all three issues by construction.

**Approved scope (must-haves):** real box model; editable text on every text
object + a font-size control; smooth drag (grab-offset) + crisp box resize;
snapping & alignment guides. **Out of scope:** per-object color/weight styling,
object duplication, keyboard nudging (not requested).

## Current state (what exists today)

- `lib/kiosk-layout.ts` — `KioskElement { id, kind: "builtin"|"text", builtin?,
  text?, visible, x, y (center fraction), sx, sy (scale multipliers), z }`,
  `normalizeKioskLayout`, `createTextElement`, `elementLabel`, `DEFAULT_KIOSK_LAYOUT`,
  `BUILTIN_IDS`, `KIOSK_ELEMENT_LABEL`, `SCALE_MIN/MAX`, `MAX_CUSTOM`, `MAX_TEXT_LEN`,
  layout-level `clockTimezone`/`clock24h`/`wifiLevel`. (+ test file.)
- `lib/kiosk-geometry.ts` — pure `resizeBox(box, handle, pointer, keepAspect)` /
  `clampCenter` over a center+`w`/`h` `Box` in canvas fractions; `HANDLES`,
  `MIN_BOX`. (+ test file.)
- `components/device-preview/kiosk-editor/use-kiosk-editor.ts` — `useKioskEditor`
  hook owning refs/selection/drag/measurement + handlers; `endInteraction`.
- `components/device-preview/kiosk-editor/kiosk-stage.tsx` — canvas + selection
  overlay (`SelectionOverlay`/`ResizeHandleDot`), right pane.
- `components/device-preview/kiosk-editor/kiosk-controls.tsx` — element list +
  `Inspector` (X/Y/W/H + text + bring-to-front) + clock/wifi + reset, left pane.
- `components/device-preview/kiosk-preview.tsx` — `KioskPreview` (7 screens) +
  exported `KioskElementView`, `ElementVisual`, `kioskRootStyle`, `KioskBrand`.
  Idle screen renders objects; `ElementVisual` switches on kind/builtin and wraps
  content in `transform: scale(sx, sy)`. Built-in Tagline text is hard-coded;
  Lane shows `storeName · lane`.
- `components/branding-editor.tsx` — two-pane studio shell + sticky save bar;
  instantiates `useKioskEditor` and wires `KioskStage`/`KioskControls`. **Stays.**
- `app/(tenant)/tenant/branding/{page.tsx,actions.ts}` — load/normalize/persist
  `kioskLayout` jsonb. **No DB change.**

## Section 1 — Object model (`lib/kiosk-layout.ts`)

```ts
type KioskObjectType = "text" | "logo" | "clock" | "wifi";

interface KioskObject {
  id: string;            // type-stable id for fixed widgets ("logo"/"clock"/"wifi"),
                         // or "text-<rand>" for text objects
  type: KioskObjectType;
  x: number; y: number;  // TOP-LEFT anchor, fraction 0..1 of the square canvas
  w: number; h: number;  // box size, fraction 0..1 of the canvas
  visible: boolean;
  z: number;             // stacking order
  // text only:
  text?: string;
  fontSize?: number;     // px on the 720 reference
  align?: "left" | "center" | "right";
}

interface KioskLayout {
  version: 2;            // bumped; v1 (sx/sy) is migrated to the default
  clockTimezone: string;
  clock24h: boolean;
  wifiLevel: number;     // 0..4
  objects: KioskObject[];
}
```

- **Fixed widgets:** exactly one each of `logo`, `clock`, `wifi` (hideable, never
  deletable). Their type config (timezone/24h/level) stays layout-level.
- **Text objects:** zero or more (cap `MAX_CUSTOM`), each with `text` (≤ `MAX_TEXT_LEN`),
  `fontSize`, `align`; fully editable + deletable. The default layout seeds two:
  a tagline line and a lane line (with the old default strings) so today's look is
  preserved and both are now editable. (Lane loses its automatic store-name binding
  — accepted.)
- **Units:** position/size stored as canvas fractions (responsive); the UI shows
  px = `fraction × 720`. `fontSize` stored as px-on-720.
- **Helpers:** `createTextObject(text, z)`; `objectLabel(o)` (type label or
  truncated text); `DEFAULT_KIOSK_LAYOUT`. Keep `MAX_CUSTOM`, `MAX_TEXT_LEN`; add
  `FONT_MIN`/`FONT_MAX`, `MIN_BOX` reuse from geometry.
- **`normalizeKioskLayout(raw)`** rewritten:
  - If `raw` is not the new shape (no `version: 2` / no `objects` array) → return a
    deep copy of `DEFAULT_KIOSK_LAYOUT` (v1 migration = reset, see §5).
  - Ensure exactly one of each fixed widget (re-add missing from defaults).
  - Keep ≤ `MAX_CUSTOM` valid text objects (`type:"text"`, non-empty string `text`,
    trimmed to `MAX_TEXT_LEN`); drop the rest and any unknown-type object.
  - Clamp `x,y` to `[0,1]`; clamp `w,h` to `[MIN_BOX, 1]`; clamp `fontSize` to
    `[FONT_MIN, FONT_MAX]`; `align` ∈ set (default `center`); `z` defaulted by order;
    timezone validated (UTC fallback); `wifiLevel` 0..4. Never throws.

## Section 2 — Geometry (`lib/kiosk-geometry.ts`)

The pure helper moves to a **top-left box** representation and gains snapping:

```ts
interface Box { x: number; y: number; w: number; h: number } // top-left, fractions
resizeBox(box, handle, pointer, opts?): Box        // opposite edge anchored, MIN_BOX floor
snapMove(box, others, threshold): { box, guides }  // snap edges/center to canvas + others
snapResize(box, handle, others, threshold): { box, guides }
```

- `resizeBox` keeps the tested edge/corner logic but on top-left boxes; free
  resize (no aspect lock) for all types; `MIN_BOX` floor; never inverts.
- **Snapping** is pure and unit-testable: given the moving box, a list of other
  boxes, and the canvas (implicit `0/0.5/1` lines), it returns the snapped box plus
  the set of active guide lines (vertical/horizontal fractions) to draw. Snap
  targets: canvas left/center/right (x: 0/0.5/1) and top/middle/bottom (y), and each
  other object's left/center/right & top/middle/bottom. Threshold in fractions.
- Replaces the old center-based `resizeBox`/`clampCenter`; both tests rewritten.

## Section 3 — Rendering (`kiosk-preview.tsx`)

`ElementVisual`/`KioskElementView` become **box-driven, no `transform: scale`**:

- Each object is an absolutely-positioned box: `left:x%, top:y%, width:w%,
  height:h%`.
- **text:** a flex box; `justify/align` per `align` (default center); `fontSize:
  cq(o.fontSize)`; `overflow-wrap: anywhere`; text wraps inside the box; no scale.
- **logo:** the uploaded image with `object-fit: contain` filling the box, or the
  brand-mark SVG sized to `min(box)` and centered — uniform, never distorted.
- **clock / wifi:** internal sizes derive deterministically from the box (e.g.
  clock time/date font and wifi bar heights computed from the box height), centered
  in the box. No DOM measurement, no `transform: scale`, no distortion. Resizing the
  box grows the graphic proportionally via its computed sizes.
- `KioskPreview` (read-only, other 6 screens) consumes the same objects so editor
  and preview never diverge.

## Section 4 — Editor (`use-kiosk-editor.ts`, `kiosk-stage.tsx`, `kiosk-controls.tsx`)

- **Hook (`useKioskEditor`)** — owns `canvasRef` (for pointer→fraction only;
  per-element DOM refs are no longer needed since the box is the source of truth and
  all sizing is deterministic), `selectedId`, `selBox` (the selected object's box in
  fractions, not a measured DOM rect), `drag` (records a **grab-offset** for moves;
  `startBox` for resizes), `guides` (active snap lines). Handlers:
  - `startMove(id, e)` records `offset = pointerFrac − object.{x,y}`.
  - `onPointerMove` (move) → `snapMove`, set `x/y` only; publish guides.
  - `startResize(handle, e)` records `startBox`. `onPointerMove` (resize) →
    `resizeBox` then `snapResize`; set `x/y/w/h`; publish guides.
  - `onPointerUp` clears drag + guides (guarded `hasPointerCapture`).
  - `addText`, `removeObject`, `bringToFront`, `resetLayout`, `endInteraction`
    (reset on stage unmount), `patch`.
  - `selBox` is derived directly from the selected object's `{x,y,w,h}` (no DOM
    measurement needed any more — the box is the source of truth).
- **`KioskStage`** — renders objects as boxes (each box owns its `onPointerDown`
  for hit-testing/selection — no measurement refs), pointer handlers from the hook,
  the selection overlay + 8 handles positioned on the selected box (in %), and the
  snap **guide lines** from the hook. Handles are crisp (fixed px), never scaled.
- **`KioskControls`** — object list (show/hide, select, delete text, Add text) +
  a **type-aware properties panel**:
  - All objects: X/Y/W/H px (draft-buffer number fields), Bring to front, delete (text).
  - text: Text content field, Font size px, alignment (left/center/right).
  - clock: timezone + 24h. wifi: level. (These write the layout-level fields.)
  - Reset layout button.
- **`branding-editor.tsx` shell, sticky save bar, Brand/Security sections:**
  unchanged except the editor now stores `objects` (the `dirty` check already
  diff's the whole `layout` via JSON).

## Section 5 — Persistence, migration, testing

- **Persistence:** same `kioskLayout` jsonb column; `saveBranding` still posts
  `kioskLayout = JSON.stringify(layout)` and normalizes server-side. **No DB
  migration.**
- **v1 → v2 migration:** old layouts use center+`sx/sy`, which can't convert to
  exact pixel boxes without runtime measurement. `normalizeKioskLayout` therefore
  **replaces any non-v2 layout with `DEFAULT_KIOSK_LAYOUT`** (a deliberate one-time
  reset, not a lossy guess). The default reproduces today's arrangement (logo,
  clock, wifi, tagline text, lane text) so the kiosk still looks right out of the box.
- **Testing:**
  - `lib/kiosk-geometry.test.ts` rewritten: top-left `resizeBox` (each edge/corner,
    MIN_BOX floor, no-invert); `snapMove`/`snapResize` (snap to canvas lines and to
    another box; guides reported; no snap outside threshold).
  - `lib/kiosk-layout.test.ts` rewritten: v2 round-trip; v1 (sx/sy) input →
    default; one-of-each fixed widget enforced; text cap/trim; box/font/align/coord
    clamps; unknown type dropped.
  - `npm run build` + `npx tsc --noEmit` clean.
  - Live verification (production, where auth works), no-save except one round-trip:
    drag has no jump (grab-offset); resize changes the box without distorting text;
    snapping guides appear and objects line up; Tagline & Lane text editable; font
    size changes; clock/wifi properties when selected; view-only disables controls.

## Out of scope (YAGNI)

- Per-object color / font-family / weight styling.
- Object duplication, multi-select, keyboard nudge/shortcuts.
- New widget types beyond text/logo/clock/wifi (no shapes/images-as-objects).
- Changes to the kiosk DB schema, the public receipt flow, or the non-idle screens'
  designs (they only gain the new object rendering for the idle preview).
