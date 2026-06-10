# Branding Page Studio Redesign — Design Spec

**Date:** 2026-06-10
**Status:** Approved design, pending implementation plan
**Goal:** Make the tenant Branding page feel like a polished product workspace, not a settings-form dump — by de-cramping the kiosk editor, regrouping controls with clear hierarchy, and fixing the save flow.

## Summary

Restructure `/tenant/branding` into a **two-pane studio** with a **sticky save
bar**, keeping every existing feature and behavior (a reskin + reorganize, not a
rewrite of functionality, and not a change to the visual design language).

Three user pain points drive this:
1. **The kiosk editor is cramped** — its controls (element list, inspector,
   clock/wifi) are squeezed into the narrow right column under the canvas.
2. **Settings-dump feel** — a flat stack of equal-weight cards with no hierarchy.
3. **Awkward save flow** — Save/Reset are buried mid-page; unsaved state isn't obvious.

The core architectural move: today's `KioskLayoutEditor` bundles the **canvas**
and its **controls** together, which is why it's cramped in a narrow column. We
split them — canvas in the right (large) pane, controls in the left pane —
sharing state via a small hook/context.

## Current state (what exists today)

- `app/(tenant)/tenant/branding/page.tsx` — server component: loads `tenant` +
  normalized `branding`, computes `canEdit`, renders `<PageHeader>` +
  `<BrandingEditor>`. **No change needed** beyond what BrandingEditor needs.
- `components/branding-editor.tsx` (client, ~470 lines) — owns ALL branding state
  (color/hex, bg/fg/muted, layout, logoText, logo file/preview/cleared, pin,
  screen, saving, dirty), the `save()`/`reset()` actions, and the
  `grid lg:grid-cols-2` layout: left = stacked `Card`s (Logo, Accent color, Kiosk
  theme, Staff PIN) + Save/Reset row; right = sticky "Live preview" `Card` with a
  screen `Select` and either `<KioskLayoutEditor>` (idle) or `<KioskPreview>`
  (other screens) inside `max-w-[420px]`. Also defines the `ColorField` subcomponent.
- `components/device-preview/kiosk-layout-editor.tsx` — `KioskLayoutEditor({ brand,
  layout, onChange, disabled })`: owns `canvasRef`, `elRefs`, `drag` ref,
  `selectedId`, `guide`, `selBox`; handlers `patch`/`pointerFrac`/`elementBox`/
  `startMove`/`startResize`/`onPointerMove`/`onPointerUp`/`addText`/`removeEl`/
  `bringToFront`; the `useLayoutEffect` that measures `selBox`; renders the canvas
  + elements + snap guides + `SelectionOverlay`, then a controls stack (element
  list, `Inspector`, Clock/Wi-Fi, Reset). Subcomponents: `SelectionOverlay`,
  `ResizeHandleDot`, `Inspector`, `NumberField`.
- `components/device-preview/kiosk-preview.tsx` — `KioskPreview` (read-only, 7
  screens) + exported `KioskElementView`/`kioskRootStyle`/`KioskBrand`. Reused by
  device dialogs. **Not changed by this redesign.**
- `lib/kiosk-layout.ts`, `lib/kiosk-geometry.ts` + tests — data model + geometry.
  **Not changed; unit tests remain valid.**
- Only consumer of `KioskLayoutEditor` is `branding-editor.tsx`.

## Section 1 — Page shell & save flow

Replace the `grid lg:grid-cols-2` with a two-pane studio:

- **Left pane** — a scrollable column of clean, well-spaced **section groups**
  (always visible, not an accordion): *Brand*, *Idle layout*, *Security*. Each has
  an icon header + one-line helper text and generous spacing. This fixes the
  "settings-dump" feel via grouping/hierarchy/breathing room without hiding anything.
- **Right pane** — a large **sticky preview stage** (the `max-w-[420px]` cap is
  raised/removed so the canvas fills the pane), screen selector on top.
- **Sticky save bar** pinned to the bottom of the page: left shows a status
  indicator — **"● Unsaved changes"** when `dirty`, **"All changes saved"**
  otherwise (or hidden when view-only); right shows **Reset** + **Save** buttons
  (same handlers/disabled logic as today: disabled when `!canEdit || saving ||
  !dirty`, Save shows the spinner while `saving`).
- **Responsive:** at `lg` the two panes sit side by side; below `lg` they stack
  (left controls above, preview below). The save bar is a sticky/fixed bottom bar
  on all sizes.
- **View-only:** the existing "view-only access" notice still renders (top of the
  left pane); all inputs stay disabled via the existing `disabled` flag.

## Section 2 — Left pane: grouped controls

Three section groups; **no feature removed or behavior changed** — only regrouped
and given space:

- **Brand** — Logo upload (keep the dashed drop-zone; once a logo is set, show a
  tidy preview chip with Replace/Remove), the logo-text fallback input, the accent
  color (swatch + hex input + preset row), and the Kiosk theme tokens
  (background / text / muted via `ColorField`) tucked into a compact "Advanced
  theme" subgroup so they don't dominate.
- **Idle layout** — the relocated editor controls: **Add text** button, the
  element list (eye toggle / click-select / trash for custom), the **Inspector**
  (X/Y/W/H px fields + text field + Bring to front), the **Clock / Wi-Fi**
  settings, and **Reset layout**. When the preview is NOT on the Idle screen, this
  section shows a subtle hint ("Switch the preview to *Idle* to edit the layout")
  and its controls are disabled. When the user switches the preview to Idle, this
  section is visually highlighted / scrolled into view.
- **Security** — Staff PIN (show/hide toggle, numeric, same behavior).

## Section 3 — Right pane: the stage

- Larger preview: raise/remove `max-w-[420px]` so the canvas fills the pane (this
  alone makes dragging far easier). Stays sticky on scroll at `lg`.
- Screen selector (`Select`, the existing 7 `SCREENS`) on top of the stage.
- Idle → `<KioskStage>` (editable canvas with drag handles + selection overlay);
  the other 6 screens → read-only `<KioskPreview>`.
- The helper caption under the stage stays (idle vs other-screen copy).

## Section 4 — Editor refactor (split canvas from controls)

Split `kiosk-layout-editor.tsx` into a shared-state module + two consumers, under
a new `components/device-preview/kiosk-editor/` directory. All current logic moves
**verbatim** — same geometry, measurement, `resizeBox`, `normalizeKioskLayout`.

- **`use-kiosk-editor.ts`** — a hook returning shared state + handlers, given
  `{ layout, onChange, disabled }`. Owns: `canvasRef`, `elRefs`, `drag` ref,
  `selectedId`/`setSelectedId`, `guide`, `selBox`, and the handlers `patch`,
  `pointerFrac`, `elementBox`, `startMove`, `startResize`, `onPointerMove`,
  `onPointerUp`, `addText`, `removeEl`, `bringToFront`, plus `selected`,
  `ordered`, `customCount`/`atCustomCap`, and the `useLayoutEffect` that measures
  `selBox`. Returns everything both consumers need. (Implemented as a hook whose
  return value is passed as a prop to both consumers, or via a React context
  provider — implementer's choice; a single instance must back both panes.)
- **`kiosk-stage.tsx`** — `<KioskStage>`: the canvas div (with `kioskRootStyle`,
  pointer handlers, snap guides), the mapped element nodes (attaching `elRefs` +
  `startMove`), and `SelectionOverlay`. Right pane.
- **`kiosk-controls.tsx`** — `<KioskControls>`: Add-text + element list +
  `Inspector` + Clock/Wi-Fi + Reset. Left pane. The `Inspector`/`NumberField`
  subcomponents move into this file; `SelectionOverlay`/`ResizeHandleDot` stay
  with the stage (`kiosk-stage.tsx`).
- **`branding-editor.tsx`** becomes the shell: instantiates the editor hook once,
  renders `<KioskControls>` inside the left *Idle layout* section and
  `<KioskStage>` in the right pane, and owns the sticky save bar. The old
  monolithic `KioskLayoutEditor` is removed (its only consumer is this file).
- Shared refs work across panes because they point at real DOM nodes; both panes
  render in the same React commit, so refs are set before layout effects run.

## Out of scope (YAGNI)

- New visual design language / theme overhaul ("looks plain" was NOT a stated
  pain point — keep shadcn cards, current typography/colors; polish via spacing
  and grouping only).
- Onboarding wizard / step flow / brand templates/presets.
- Any change to the kiosk data model, geometry, persistence, or the public
  receipt/kiosk rendering.
- Changes to `KioskPreview` internals or the device-dialog reuses of it.
- Keyboard shortcuts (e.g. Cmd+S) — nice-to-have, not required.

## Testing

- `lib/kiosk-geometry.test.ts` and `lib/kiosk-layout.test.ts` are untouched and
  must still pass (the refactor moves rendering, not logic).
- `npm run build` + `npx tsc --noEmit` clean.
- Live verification (in production, where auth works): the two-pane layout renders;
  the kiosk editor's controls sit in the left pane and the canvas in the right;
  select/drag/resize/add/delete/inspector-typing all still work via the split
  components; the sticky save bar shows unsaved state and Save/Reset behave; switching
  preview screens disables/enables the Idle-layout section; view-only mode disables
  everything. (No-save dogfooding to avoid mutating the demo tenant, then one
  deliberate save+reload to confirm persistence still round-trips.)
