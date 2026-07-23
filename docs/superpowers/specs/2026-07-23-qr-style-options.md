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
