# Printer Screen Flexibility — Clock Options, Top-Bar Preset, Cleaner Default

**Date:** 2026-06-14
**Status:** Approved (brainstorming) → ready for implementation plan

## Summary

Three focused improvements to the printer branding editor's per-screen object model (the v3 `PrinterConfig` in `lib/printer-layout.ts`):

1. **Cleaner idle default** — stop seeding the "Lane 1" and "Tap your card…" placeholder text labels.
2. **Per-clock display options** — let a tenant hide the date line, drop the weekday name, and align the clock text (so it can sit flush top-left), in addition to the existing shared timezone / 12-24h.
3. **"Insert top bar" preset** — a one-click button that arranges logo / clock / wifi into a tidy top row, then the tenant tweaks freely.

All data lives in the existing `tenant_settings.kiosk_screens` jsonb (the v3 `PrinterConfig`). **No DB migration** — `normalizePrinterConfig` supplies defaults for the new field, so existing configs keep working and keep their current look.

## Context (current state)

- Screens are object-based: each `PrinterScreen` has a `ScreenLayout { objects: PrinterObject[] }`. `logo`/`clock`/`wifi`/`qr`/… are **singleton widgets** (≤1 per screen, hideable, not user-addable); `text`/`icon` are user-addable. Everything is a positionable box (`x/y/w/h` fractions of the 720² canvas), so the clock is **already draggable** anywhere.
- `seededText()` (`lib/printer-layout.ts`) seeds two text objects — `text-lane` ("Lane 1") and `text-tagline` ("Tap your card or pay at the reader to begin") — into the idle default (`defaultLayout()` → idle seed).
- `PrinterClock` (`components/device-preview/printer-clock.tsx`) always renders the time plus a date line formatted `{ weekday:"long", month:"long", day:"numeric" }` (e.g. "Saturday June 14"), text-align center hard-coded. Only `clockTimezone` and `clock24h` (shared, config-level) are configurable.
- `ClockObject` (in `printer-preview.tsx`) renders `<PrinterClock timezone clock24h size />`.
- Clock controls live in `PrinterControls` (`printer-editor/printer-controls.tsx`) Properties: a timezone `Select` + a 24h `Switch`.
- The editor hook `usePrinterEditor` exposes per-screen actions (`addText`, `addIcon`, `patch`, `setShared`, `resetLayout`, …) operating on the active screen's objects.

## Design

### 1. Cleaner idle default

- `seededText()` returns `[]` (remove both label objects). `defaultLayout()` then seeds only the logo/clock/wifi widgets; `seededScreen("idle")` (which delegates to `defaultLayout().objects`) inherits this.
- **Scope:** affects the built-in default only. Existing tenants' saved configs are untouched (those labels remain until deleted in the editor) — `normalizePrinterConfig` never re-adds them.
- Update `printer-layout.test.ts`: the idle seed test should assert **0 text objects** (was 2), logo/clock/wifi still present.

### 2. Per-clock display options

**Data model (`lib/printer-layout.ts`):**
- Add an optional clock sub-config to `PrinterObject`, mirroring the existing `icon?` pattern:
  ```ts
  export interface PrinterClockOptions {
    showDate?: boolean;    // default true — the whole second line
    showWeekday?: boolean; // default true — the day name within the date
  }
  // on PrinterObject:
  clock?: PrinterClockOptions;
  ```
- Reuse the existing `align?: TextAlign` field for the clock's text alignment (no new field).
- Defaults preserve today's look: missing `clock` ⇒ `{ showDate: true, showWeekday: true }`; missing `align` on a clock ⇒ `"center"`.

**Normalize (`normalizePrinterConfig` → `sanitizeObject`):**
- For `type === "clock"`, in addition to the box, carry:
  - `align`: validated against `ALIGNS`, default `"center"`.
  - `clock`: `{ showDate: bool(default true), showWeekday: bool(default true) }` via a small `sanitizeClock(raw)` helper (mirrors `sanitizeIcon`). Unknown/garbage ⇒ defaults. Never throws.

**Renderer (`PrinterClock` + `ClockObject`):**
- `PrinterClock` gains `showDate = true`, `showWeekday = true`, `align: TextAlign = "center"` props.
  - When `!showDate`: omit the date `<div>` entirely (time only).
  - When `showDate && !showWeekday`: format date as `{ month:"long", day:"numeric" }` (e.g. "June 14"); else keep `{ weekday:"long", month:"long", day:"numeric" }`.
  - `align` drives `textAlign` on the root and, since the block is centered today, also the horizontal placement (root `style.textAlign = align`).
- `ClockObject` passes `object.clock?.showDate`, `object.clock?.showWeekday`, and `object.align` through to `PrinterClock`.
- Shared `config.clockTimezone` / `config.clock24h` are unchanged.

**Editor UI (`PrinterControls` clock Properties):**
- Keep the timezone `Select` + 24h `Switch`.
- Add: **Align** (left/center/right buttons → `editor.patch(id, { align })`, reusing the text-align control pattern), **Show date** `Switch` (→ `patch(id, { clock: { ...current, showDate } })`), **Show weekday** `Switch` (→ `patch(id, { clock: { ...current, showWeekday } })`, disabled when `showDate` is false).

### 3. "Insert top bar" preset

**Helper (`lib/printer-layout.ts`):** `topBarArrangement()` returns the boxes + clock settings for a tidy top row (approximate; tunable during the build smoke-check):
- `logo`: `{ x: 0.04, y: 0.04, w: 0.30, h: 0.10 }` (left)
- `clock`: `{ x: 0.40, y: 0.045, w: 0.20, h: 0.09 }`, `align: "center"`, `clock: { showDate: false, showWeekday: true }` (compact, time-only)
- `wifi`: `{ x: 0.86, y: 0.05, w: 0.10, h: 0.06 }` (right)

**Editor action (`usePrinterEditor`):** `insertTopBar()` operates on the active screen — for each of logo/clock/wifi, upsert the singleton (create if absent, else update) with its preset box (and clock options), `visible: true`, layered above existing objects. Other objects on the screen are untouched. Returns via the normal `onChange(config)`.

**UI:** an **"Insert top bar"** button in `PrinterControls` near "Add text"/"Add icon" (acts on the active screen). After inserting, the three widgets are normal objects the tenant can drag/restyle.

## Components & boundaries

| Unit | Change |
|---|---|
| `lib/printer-layout.ts` | `seededText()` → `[]`; add `PrinterClockOptions` + `clock?` on `PrinterObject`; `sanitizeClock` + clock branch in `sanitizeObject` (align + clock); `topBarArrangement()` helper. |
| `lib/printer-layout.test.ts` | idle-seed assertion (0 text); `normalizePrinterConfig` clock-sanitize defaults + bad input; `topBarArrangement` shape. |
| `components/device-preview/printer-clock.tsx` | `showDate`/`showWeekday`/`align` props + conditional date formatting. |
| `components/device-preview/printer-preview.tsx` | `ClockObject` passes the new clock props through. |
| `components/device-preview/printer-editor/use-printer-editor.ts` | `insertTopBar()` action (+ on the `PrinterEditor` interface). |
| `components/device-preview/printer-editor/printer-controls.tsx` | clock Properties: Align + Show date + Show weekday; "Insert top bar" button. |

## Data flow & persistence

- New `clock` field rides inside each clock object in the v3 `PrinterConfig` JSON, persisted to `tenant_settings.kiosk_screens` (physical column unchanged). `getTenantBranding` → `normalizePrinterConfig` fills defaults on read; `saveBranding` persists the config as-is (already normalizes). **No schema/DB migration.**
- Back-compat: an existing clock object with no `clock`/`align` renders exactly as today (defaults true/center).

## Testing

- **Pure (vitest):** idle seed has no text objects; `normalizePrinterConfig` defaults `clock` to `{showDate:true,showWeekday:true}` and coerces bad input; clock `align` validated; `topBarArrangement()` returns logo/clock/wifi boxes on-canvas with the compact clock settings.
- **UI smoke (dev server):** select the clock → toggle Show date / Show weekday / Align and see the preview update; click "Insert top bar" → logo/clock/wifi snap into a top row; save → reload persists; the idle default (fresh tenant) shows no placeholder labels.

## Out of scope (YAGNI)

- No shared/linked "header component" that auto-syncs across screens (rejected in favor of flexible objects + a preset).
- No seconds in the clock, no custom date-format strings, no per-object opacity/badge styling (separate future work).
- No change to shared `clockTimezone`/`clock24h` semantics.
- Existing tenants' "Lane 1"/tagline are not bulk-removed (deletable in-editor).
