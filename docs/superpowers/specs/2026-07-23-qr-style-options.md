# QR Style Options (shape + colors) — cloud side

*2026-07-23 · approved in conversation. FW counterpart: ditto-firmware
`docs/superpowers/specs/2026-07-23-qr-style-options-firmware.md` (0.11.0).*

## Decisions (locked)
- **Org-wide setting** (one brand = one QR look), stored INSIDE the printer config
  JSON top-level (next to `clockTimezone`/`qrTimeoutSeconds`): no DB migration;
  normalize supplies defaults; ETag covers changes automatically.
- Fields: `qrShape: "classic" | "soft" | "rounded" | "dots"` (default `"rounded"` —
  today's live look), `qrFg: "#rrggbb"` (default `"#111111"`), `qrBg: "#rrggbb"`
  (default `"#ffffff"`).
- **Scannability guardrails (normalize-enforced, server-side):** fg must be DARKER
  than bg (relative luminance) AND WCAG-style contrast ratio ≥ 4:1; invalid pair →
  both reset to defaults. Unknown shape → `"rounded"`. Malformed hex → default.
  Dots shape keeps dot diameter ≥ 0.7×module (helper already clamps).
- Applies to every QR render: device screens (trigger + pinned + setup), studio
  previews, device-pin-control card. One shared style type exported from
  lib/printer-layout.ts.

## Work
1. `lib/printer-layout.ts`: `qrShape/qrFg/qrBg` in the config type + normalize
   (defaults, shape whitelist, hex validation, contrast guard as a PURE exported
   helper `sanitizeQrStyle(...)` with unit tests: valid pass-through, unknown
   shape, bad hex, low contrast, inverted colors → defaults).
2. `lib/qr-svg.ts` / `components/qr-svg.tsx`: accept `{shape, fg, bg}`; shape
   variants — classic: squares; soft: rounded-corner squares (rx≈0.25×module);
   rounded: current circles+rounded finders; dots: circles d=0.7×module. Bg rect
   painted (incl. quiet-zone area).
3. Branding studio: "QR style" section (Branding page, near theme tokens): 4-way
   shape picker showing mini previews + fg/bg color inputs (reuse existing color
   input pattern) + inline warning when contrast guard would reset. Saved through
   the existing branding save action (config JSON path).
4. Device config payload: fields flow with normalized config automatically —
   verify they appear top-level for firmware and are ETag-covered (test).
5. device-pin-control + printer-preview: consume the org style (they receive or
   can receive branding/config — thread minimal props).
6. Gates: vitest/tsc/lint/build.

## Out of scope
Per-object styles, gradients, logos-inside-QR, inverted (light-on-dark) QRs.

---

## Addendum: background corner + shadow (2026-07-23, branch `feat/qr-corner-shadow`)

**Bug report:** the branding studio preview showed the QR's background plate
with rounded corners and a soft drop-shadow; the device renders it square and
flat. Root cause — that rounding/shadow was hard-coded chrome in two React
call sites, not derived from any config:
- `components/device-preview/printer-preview.tsx` `QrObject` — the wrapper
  `<div>` around every QR render (idle/qr/setup screens in the studio
  canvas + filmstrip) always set `borderRadius: cq(14|32)` and, for the
  non-compact case, a fixed `boxShadow`.
- `components/device-pin-control.tsx` — the pinned-QR `<QrSvg>` on the tenant
  device detail page always carried the Tailwind class `rounded-lg` (no
  shadow there before this change).

**Decisions (locked):** same org-wide model as shape/fg/bg — two more fields
alongside them in the same top-level config JSON block, no migration:
- `qrCorner: "square" | "rounded"` (default `"rounded"` — preserves today's
  look for existing tenants).
- `qrShadow: boolean` (default `false` — matches the device's actual
  shadowless render; existing tenants get the corrected default, not the old
  hard-coded `true`).
- Both sanitize independently and permissively: unknown/malformed `qrCorner`
  → `"rounded"`; non-boolean `qrShadow` → `false`. No cross-field validation
  (unlike the retired fg/bg contrast guard — these have no scannability
  implication).

**Work done:**
1. `lib/printer-layout.ts` — `QrStyle` + `PrinterConfig` gain `qrCorner`/
   `qrShadow`; `DEFAULT_QR_STYLE` extended; `sanitizeQrStyle` coerces both.
   `normalizePrinterConfig`/`migrateV2ToConfig` needed no further change —
   both already spread `sanitizeQrStyle(cfg)` / `DEFAULT_QR_STYLE`. Unit
   tests: defaults for garbage/missing input, valid pass-through, unknown
   corner → `"rounded"`, non-boolean shadow → `false`, explicit `false` kept,
   plus `normalizePrinterConfig` defaults/pass-through/v2-migration/round-trip
   coverage mirroring the existing shape/fg/bg tests.
2. `lib/qr-svg.ts` — new `QR_CORNERS`/`QrCorner` + pure `qrBackgroundRadius(dim,
   corner)` helper (~6% of `dim` when rounded, 0 when square).
   `components/qr-svg.tsx` — `QrSvg` gains `corner`/`shadow` props; the
   background `<rect>` gets `rx={qrBackgroundRadius(...)}` and, when
   `shadow`, an SVG `<filter>` (`feDropShadow`) referenced via a
   collision-safe id (`useId()` with `:` stripped — raw `useId()` output
   isn't a legal bare token inside `url(#...)`).
3. **Fix for the reported bug** — both hard-coded call sites now read the org
   setting instead of a constant:
   - `printer-preview.tsx` `QrObject`: `borderRadius`/`boxShadow` are now
     `config.qrCorner === "rounded" ? … : 0` / `config.qrShadow ? … :
     undefined`. The inner `<QrSvg>` gets `corner={config.qrCorner}` (for
     architectural consistency — every `QrSvg` render reflects the real
     style) but `shadow={false}` explicitly, so the shadow is painted once
     (at the wrapper) rather than doubled.
   - `device-pin-control.tsx`: the previously-static `rounded-lg` class is
     now `cn(qrCorner === "rounded" ? "rounded-lg" : "rounded-none", qrShadow
     && "shadow-md")`; new `qrCorner`/`qrShadow` props threaded down from the
     device detail page's existing `getOrgQrStyle()` call (`qrStyle.qrCorner`
     / `qrStyle.qrShadow`), same pattern as the existing `qrShape`/`qrFg`/
     `qrBg` props.
4. Branding studio — QR controls moved out of the Theme tab into their own
   tab (`ControlPanel`'s `Tabs`: Theme / **QR** / Screen, `QrCode` icon,
   between the other two — follows the existing `TabsTrigger`/`TabsContent`
   pattern already used for Theme/Screen). `QrStylePanel` now renders two
   sections: "Shape" (unchanged shape swatches + fg/bg color fields) and a
   new "Background plate" section — a 2-up square/rounded swatch picker
   (mirrors the 4-up shape-swatch pattern) plus a `Switch` for "Drop shadow"
   (mirrors the existing screen-colors-override switch pattern).
5. `lib/device-config.test.ts` — extended the existing "computeConfigVersion
   — QR style" describe block with a case for `qrCorner` changing within
   `printerScreens` (mirrors the existing qrShape/qrFg/qrBg case; both fields
   live top-level inside the same JSON blob with no dedicated ETag input, so
   they're covered by the "any renderable input changes" test regardless —
   this pins one of the two new fields down explicitly, same rationale as
   the original qrShape/qrFg/qrBg case).

**Gates:** `tsc --noEmit` 0 errors; `vitest run` 373/373 (was 372, +9 new
qrCorner/qrShadow cases, 1 pre-existing `toEqual` snapshot updated for the
2 new `QrStyle` fields); `next build` clean; `eslint` — identical 6
pre-existing problems (0 new), diffed against the pre-branch tree.

**Out of scope:** the firmware counterpart (device-side corner/shadow
rendering to actually vary, not just match today's square/flat default) —
tracked separately, same split as the original spec's firmware doc.

---

## Addendum: shadow intensity + color + neon effect (2026-07-24, branch `feat/qr-shadow-neon`)

**Product ask:** the boolean `qrShadow` on/off from the addendum above was too
blunt — add an intensity slider, a shadow color, and a second "neon glow"
effect alongside the plain drop shadow.

**Decisions (locked):** replaces the boolean field with a 3-way mode + two
tunables, same top-level config-JSON block, still no DB migration:
- `qrShadowMode: "none" | "drop" | "neon"` (default `"none"`). Sanitize
  migrates legacy stored data: a valid `qrShadowMode` always wins; otherwise a
  legacy boolean `qrShadow: true` → `"drop"`, `false`/absent/anything else →
  `"none"` (the new default — same as the old boolean default).
- `qrShadowStrength: 0..100` int (default `50`) — clamp + round, reused
  `clamp`/`num` helpers already in `lib/printer-layout.ts`.
- `qrShadowColor: "#rrggbb"` (default `"#000000"`) — reuses the existing
  `normalizeHex` hex normalizer (malformed → default, independent of the
  other two fields).
- No cross-field constraints, matching the corner+shadow addendum's stance —
  permissive, the merchant's call.
- The old `qrShadow: boolean` field is **removed from the `QrStyle`/
  `PrinterConfig` types** (TypeScript no longer sees it); `sanitizeQrStyle`
  still *reads* a raw `qrShadow` key off the untyped input for the one-time
  migration path above.

**Work done:**
1. `lib/qr-svg.ts` — new `QR_SHADOW_MODES`/`QrShadowMode`, plus the shared
   shadow math consumed by every renderer:
   - `qrShadowBoxShadow(mode, strength, color)` → CSS `box-shadow` string (or
     `undefined` for `"none"`) for the two wrapper-`<div>` previews. Drop =
     `0 2px {blur}px {rgba(color, opacity)}` (opacity `0.25 + 0.5×(strength/
     100)`); neon = `0 0 {blur}px {color}, 0 0 {blur×2}px {color}` — two
     stacked zero-offset full-color glows, the "classic neon halo" look.
   - `qrShadowFilterSpec(mode, strength, color)` → parameters for the SVG
     `<filter>` QrSvg builds (`null` for `"none"`). Drop = one
     `feDropShadow` (`dy≈2`, `stdDeviation` ∝ strength, ≈1..12, pre-mixed
     `floodColor` rgba). Neon = `feFlood` + `feComposite in2="SourceAlpha"`
     to recolor the background rect's own alpha into `color` (so the glow
     isn't tinted by the QR's own fg/bg), two `feGaussianBlur` passes (tight
     + wide, both ∝ strength) merged back under `SourceGraphic` — an
     offset-0 glow, not a cast shadow.
2. `lib/printer-layout.ts` — `QrStyle`/`PrinterConfig` gain `qrShadowMode`/
   `qrShadowStrength`/`qrShadowColor`, lose `qrShadow`. `DEFAULT_QR_STYLE`
   updated. `sanitizeQrStyle` gains `sanitizeQrShadowMode` (valid stored mode
   wins; else legacy boolean migration; else default) plus clamp/hex handling
   for strength/color. `normalizePrinterConfig`/`migrateV2ToConfig` needed no
   further change (both already spread `sanitizeQrStyle(cfg)` /
   `DEFAULT_QR_STYLE`).
3. `components/qr-svg.tsx` — `QrSvg`'s `shadow?: boolean` prop replaced with
   `shadowMode?`/`shadowStrength?`/`shadowColor?`; the `<defs>` block now
   builds either the `feDropShadow` or the `feFlood`/`feComposite`/
   `feGaussianBlur`×2/`feMerge` chain per `qrShadowFilterSpec`'s discriminated
   result, and the background `<rect>`'s `filter` attribute follows suit.
4. Preview wrappers switched from the old CSS class/hard-coded `boxShadow` to
   the shared `qrShadowBoxShadow` helper, and stopped double-applying the
   shadow at the inner `QrSvg` (which now defaults its shadow prop to
   `"none"`, so omitting it is enough):
   - `components/device-preview/printer-preview.tsx` `QrObject` — one
     `boxShadow` computed from `config.qrShadowMode/qrShadowStrength/
     qrShadowColor`, used for both the compact and full-size wrapper (the
     previous two hand-tuned constants collapsed into the one shared
     formula, per the locked design).
   - `components/device-pin-control.tsx` — the previous `qrShadow &&
     "shadow-md"` Tailwind toggle replaced with an inline `boxShadow` from
     the same helper; new `qrShadowMode`/`qrShadowStrength`/`qrShadowColor`
     props threaded from the device detail page's existing `getOrgQrStyle()`
     call, replacing the single `qrShadow` prop.
   - `components/device-preview/printer-editor/use-printer-editor.ts` —
     `setShared`'s field whitelist swaps `"qrShadow"` for the three new keys.
5. Branding studio (`components/branding-studio/branding-studio.tsx`)
   `QrStylePanel`'s "Background plate" section: the `Switch` "Drop shadow"
   toggle is replaced by a 3-up mode picker (None/Drop/Neon — mirrors the
   existing 2-up corner-swatch pattern, each swatch's `boxShadow` computed
   live via `qrShadowBoxShadow` so the picker previews the actual current
   strength/color) and, only when the mode isn't `"none"`, an "Intensity"
   `Slider` (0–100, existing `ui/slider` component — same pattern as the
   device-settings brightness/QR-timeout sliders) plus a "Shadow color"
   `ColorField` (the existing fg/bg color-input component, reused as-is).
6. Tests:
   - `lib/printer-layout.test.ts` — full sanitize matrix: defaults for
     garbage/missing input, valid pass-through, unknown mode → `"none"`,
     legacy `qrShadow: true`→`"drop"` and `false`/absent→`"none"` migration
     both directions, a valid `qrShadowMode` beating a conflicting legacy
     `qrShadow`, strength clamp (150→100, -5→0) + rounding + non-numeric
     fallback, malformed/shorthand shadow-color hex — plus the
     `normalizePrinterConfig` top-level default/pass-through/v2-migration/
     legacy-migration/round-trip coverage mirroring the existing corner
     tests.
   - `lib/qr-svg.test.ts` — `qrShadowFilterSpec`: null for `"none"`, correct
     `"drop"`/`"neon"` discriminated shape, monotonic stdDeviation/blur growth
     with strength. `qrShadowBoxShadow`: undefined for `"none"`, drop/neon
     CSS shape (neon's second glow radius ≈ 2× the first), distinct output
     per mode.
   - `lib/device-config.test.ts` — extended the "computeConfigVersion — QR
     style" describe block with a case for `qrShadowMode`/`qrShadowStrength`/
     `qrShadowColor` changing within `printerScreens` (mirrors the existing
     qrShape/qrFg/qrBg/qrCorner cases — same "covered by the whole blob"
     reasoning, pinned down explicitly for the new fields).

**Gates:** `tsc --noEmit` 0 errors; `vitest run` 393/393 (was 373, +20 net
new across the three touched test files); `next build` clean; `eslint` —
same 6 pre-existing problems in the two touched component files (0 new),
diffed line-for-line against the pre-branch tree.

**Out of scope:** the firmware counterpart (device-side shadow/glow
rendering) — tracked separately, same split as the prior two addenda.
