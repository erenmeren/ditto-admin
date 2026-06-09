# Flexible Kiosk Studio — Design Spec

**Date:** 2026-06-09
**Branch:** `kiosk-preview`
**Status:** Approved design, pending implementation plan

## Summary

Turn the Kiosk Studio idle-screen editor from a fixed-element tool into a
freeform, design-tool-like editor. Three new capabilities:

1. **Add / remove elements** — keep the 5 built-ins (logo, clock, wi-fi, lane,
   tagline) as show/hide elements, and let users add their own **custom text
   boxes** (type any text, then position/size it).
2. **Pixel-precise number inputs** — type exact **X, Y, W, H in pixels** (on the
   720×720 design canvas) into number boxes, two-way bound with the canvas.
3. **MS-Word-style resize handles** — select an element to get 8 handles
   (4 corners + 4 edges). Corners scale proportionally; edges **free-stretch**
   one axis (text distorts, intended).

Chosen technical approach: **Approach 1 — bounding-box + fit-to-content
transform.** It is the only model that delivers pixel inputs *and* uniform
free-stretch across every element type (text and the logo/wi-fi vectors).

## Current state (what exists today)

- `lib/kiosk-layout.ts` — `KioskLayout` with a closed set of 5 element ids;
  each element is `{ id, visible, x, y, scale }` with fractional `x,y` (0..1
  center anchor) and a single `scale` (0.5–2). `normalizeKioskLayout` coerces
  stored jsonb into a valid layout. `DEFAULT_KIOSK_LAYOUT` seeds positions.
- `components/device-preview/kiosk-preview.tsx` — `KioskPreview` renders the 7
  kiosk screens; the idle screen maps visible elements to absolutely-positioned
  `KioskElementView`s. `KioskElementView` switches on element id and renders the
  visual at a size driven by `scale`. `kioskRootStyle` exposes brand CSS vars.
  Shared by the preview and the editor so they never drift.
- `components/device-preview/kiosk-layout-editor.tsx` — drag-to-position on a
  fractional canvas with snap-to-center guides; a controls rail with per-element
  eye toggle + a scale **slider**; clock timezone / 24h; wi-fi level; reset.
- `components/device-preview/kiosk-clock.tsx` — live timezone clock.
- `components/branding-editor.tsx` — owns `layout` state, dirty detection, reset,
  and serializes `layout` into the save `FormData`.
- `app/(tenant)/tenant/branding/{page.tsx,actions.ts}` — loads branding +
  normalized layout, persists to `tenantSettings` jsonb. No new migration needed.

Limitations being removed: position is drag-only, size is slider-only, the
element set is fixed at 5, and there is no way to type exact values.

## Section 1 — Data model (`lib/kiosk-layout.ts`)

Replace the element shape and open up the id set.

```ts
type BuiltinId = "logo" | "clock" | "wifi" | "lane" | "tagline";

interface KioskElement {
  id: string;            // builtin id, or "text-<nanoid>" for custom
  kind: "builtin" | "text";
  builtin?: BuiltinId;   // present when kind === "builtin"
  text?: string;         // present when kind === "text"
  visible: boolean;
  x: number; y: number;  // CENTER anchor, fraction 0..1 (unchanged meaning)
  w: number; h: number;  // box size, fraction of the 720 canvas (replaces scale)
  z: number;             // stacking order for overlapping elements
}
```

Rules:

- **Storage stays fractional** (0..1). The responsive canvas, jsonb column,
  presigned-URL flow, and normalize/validation infra are untouched. The UI
  multiplies by 720 to show/edit pixels and divides on write.
- **`x,y` remain the element CENTER** (not top-left), matching today's math.
- **`scale` → `w,h` migration:** a per-builtin `DEFAULT_BOX` table (fractions,
  captured from today's natural rendering) × the legacy `scale`. Existing saved
  layouts keep their look. Custom elements get a default box.
- **`normalizeKioskLayout` reworked** for an open id set:
  - Built-ins reconciled against the 5 known ids (missing ones re-added, as today).
  - Unknown ids kept only if `kind:"text"` with a string `text`; otherwise dropped.
  - `w,h` clamped to a sane range (floor ≈ 0.03, ceiling 1.0); `x,y` clamped so
    the center stays in 0..1; `z` defaulted by array order.
  - Legacy elements lacking `kind`/`w`/`h` are migrated (kind = builtin, box from
    `DEFAULT_BOX × scale`).
- Constants: replace `SCALE_MIN/SCALE_MAX` usage with box min/max; keep
  `KIOSK_ELEMENT_LABEL` for built-ins and derive a label for custom text
  (e.g. the text content, truncated, or "Text").

## Section 2 — Editor UX (`kiosk-layout-editor.tsx`)

Add a real selection + inspector model.

- **Select** an element (click) → 8 resize handles appear (4 corners + 4 edges),
  body shows a move cursor. Click empty canvas deselects.
- **Drag body** → move; keep the existing snap-to-center guides.
- **Drag corner** → resize proportionally (lock w:h ratio).
- **Drag edge** → stretch that single axis (free stretch; text distorts — intended).
- **Inspector panel** for the selected element:
  - Number boxes for **X, Y, W, H in pixels** (0–720), two-way bound with handles.
  - Per-kind controls: text-content field (custom text), timezone + 24h (clock),
    wi-fi level (wifi), show/hide, bring-to-front / send-to-back (`z`), and
    **delete** (custom text only).
- **"+ Add text" button** → creates a `text` element at center, selected, ready
  to edit its content.
- The element-list rail stays as the show/hide + quick-select list: built-ins
  keep the eye toggle; custom text elements get a trash icon. Reset-to-default
  unchanged.

Decisions locked: X/Y are the element **center**; custom text is **deletable**,
built-ins are only **hideable** (logo/clock can never be lost).

## Section 3 — Rendering (`kiosk-preview.tsx`)

Approach 1's fit-to-content, shared by preview and editor.

- **`KioskElementView` becomes box-driven.** Each element is absolutely
  positioned at its `{x,y}` center with pixel box `{w,h}`. Inside, the existing
  visual (logo SVG, `KioskClock`, wi-fi bars, lane/tagline, or custom text)
  renders at its **natural size** in a measuring wrapper, then gets
  `transform: scaleX(w/naturalW) scaleY(h/naturalH)` with
  `transform-origin: center`.
- **Measurement hook `useNaturalSize`** — a `ref` + `useLayoutEffect` /
  `ResizeObserver` reporting the content's natural pixel size within the
  container. Natural size and box share the same container units, so the ratio is
  resolution-independent — the canvas can be any width and it holds.
- **First-render / default box** — when an element has no stored box (brand-new
  custom element, or a built-in being seeded), measure natural size once and
  write it back as the box, so "scale 1" == natural. Thereafter the stored box is
  the source of truth.
- **Custom text** renders with the kiosk font, `--k-fg` color, centered,
  `white-space: pre` so the box is the wrap authority (stretching the box changes
  it, matching the free-stretch model).
- The editor reuses this exact view; handles are an overlay around the box, so
  what you drag is pixel-identical to what renders on the real kiosk.

## Section 4 — Persistence, edge cases, testing

- **Persistence:** unchanged plumbing. `branding-editor.tsx` already serializes
  `layout` into the save `FormData`; the richer object rides the same path.
  `actions.ts` stores jsonb; `normalizeKioskLayout` guards every load. **No new
  DB migration** — the column already holds jsonb.
- **Edge cases:**
  - Min box-size floor so an element can't be dragged to 0×0 and become
    unselectable.
  - Clamp boxes so an element can't be dragged fully off-canvas (center stays in
    0..1).
  - Pointer-capture cleanup on `pointerup` / `pointerleave` (as today).
  - Read-only preview never shows handles / selection.
  - `z`-order ties broken by array index.
- **Testing** (extends `lib/kiosk-layout.test.ts`):
  - Migration of a legacy `scale`-only layout → correct `w,h`.
  - Round-trip normalize of a custom text element.
  - Unknown / garbage element dropped.
  - Box clamping (floor/ceiling, off-canvas center).
  - Missing built-in re-added.
  - Live verification of measurement + handle drag/stretch via the
    Playwright/browse tooling against the running editor (same as today's drag
    verification).

## Out of scope (YAGNI)

- Rotation, opacity, per-element color/font overrides.
- Multi-select / group transforms.
- Snap-to-other-elements (only center guides, as today).
- Image/icon upload as custom elements (text only for now).
- Undo/redo stack (reset-to-default remains the escape hatch).
