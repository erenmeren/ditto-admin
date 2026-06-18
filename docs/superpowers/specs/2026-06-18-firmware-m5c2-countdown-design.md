# M5c-2: configurable receipt-screen countdown + auto-return

**Date:** 2026-06-18
**Status:** Approved (design)
**Repos:** ditto-admin (config + editor + preview) + ditto-firmware (widget + state)
**Parent:** `2026-06-15-device-config-driven-ui-design.md` (M5c deferred `countdown`); continues M5c-1 (shipped).

## Problem

After a receipt is ingested the device shows the QR/receipt screen (`DEV_QR`) and
**stays there indefinitely** until the next tap — there is no auto-return to
idle. The branding receipt screen includes a `countdown` object ("Code expires
0:48" + a depleting progress bar), but it renders as a placeholder on the device
and the admin preview is a static mockup with hardcoded values. M5c-2 makes the
countdown live and uses it to auto-dismiss the QR screen, so the printer resets
for the next customer.

## Scope

A live countdown widget on the receipt/QR screen **and** an auto-return
(`DEV_QR` → `DEV_IDLE` when it reaches zero). The display duration is
**tenant-configurable** in the branding editor.

## Decisions

1. **`qrTimeoutSeconds` is a new top-level shared field** on `PrinterConfig`
   (alongside `clockTimezone`/`clock24h`/`wifiLevel`). Default **60**, clamped
   **15–180**. No config version bump — `normalizePrinterConfig` defaults it when
   absent. It flows in the device payload and the config version hash
   automatically (it lives inside `printerScreens`).
2. **Auto-return is scoped to `DEV_QR` → `DEV_IDLE`.** Other screens that happen
   to carry a countdown object just render it (no state effect).
3. **UI/state separation.** The countdown is an LVGL widget that ticks precisely
   on a 1 Hz timer; the *state transition* is owned by the state machine, which
   consumes an "expired" signal from the UI (`ui_consume_countdown_expired()`).
   The UI never changes device state directly (mirrors the existing
   `ui_consume_tap()` seam).
4. **Prompt return.** While in `DEV_QR` the poll loop checks the expired flag on a
   short (~500 ms) tick so the return happens within ~1 s of 0:00, not a full
   poll cycle.

## Components

### Admin (`ditto-admin`)

1. **`lib/printer-layout.ts`** — add `qrTimeoutSeconds: number` to `PrinterConfig`
   (version stays 3); set it in `defaultLayout()`/the config default to `60`;
   parse + clamp (15–180) in `normalizePrinterConfig`. (PrinterLayout v2 stays as
   is — it's the rollback shape; the field is config-level.)
2. **Editor** — a "Receipt screen timeout" control (number input or slider,
   15–180 s) in the shared-controls section of
   `components/device-preview/printer-editor/printer-controls.tsx`, next to the
   wifi level / clock controls, wired through the editor's `setShared`.
3. **Preview** — `CountdownObject` in `components/device-preview/printer-preview.tsx`
   shows the configured duration as its starting `M:SS` (replace the hardcoded
   `remain = "0:48"` / `progress = 0.34` with the configured full value, static —
   it does not tick in the editor).
4. **Tests** — `lib/printer-layout.test.ts`: `normalizePrinterConfig` defaults the
   field to 60 when absent and clamps out-of-range values into 15–180.

### Firmware (`ditto-firmware`)

5. **`components/devcfg/device_config.h` + `cfg_parse.c`** — add
   `int qr_timeout_seconds;` to `device_config_t`; parse `"qrTimeoutSeconds"`
   (default 60, clamp 15–180). Host test in `tools/cfg-harness`.
6. **`components/devcfg/clock_format.{c,h}`** — add a pure
   `format_mmss(int seconds, char *out, int outlen)` → `"M:SS"` (e.g. 48→`0:48`,
   125→`2:05`, 0→`0:00`); host-tested next to `format_clock`.
7. **`components/ui/ui.c`** — `render_countdown`: a "Code expires" label + an
   `M:SS` remaining label + an `lv_bar` progress, registered like the clock; a
   1 Hz LVGL timer decrements from a deadline, updates label + bar, and on
   reaching 0 sets an internal expired flag. `ui_set_countdown(int seconds)`
   (re)starts it; `ui_consume_countdown_expired()` returns+clears the flag.
   Cleared/torn down per render under the LVGL lock (same registry pattern as the
   clock timer). `ui.h` declares both new functions.
8. **`main/app_state.c`** — when transitioning to `DEV_QR` (the two sites that
   currently do `ui_set_qr_url(url); ui_render_state(DEV_QR)`), also call
   `ui_set_countdown(cfg->qr_timeout_seconds)`. In `poll_task`, while
   `s_state == DEV_QR`, use a short delay and check
   `ui_consume_countdown_expired()`; on expiry set `s_state = DEV_IDLE;
   ui_render_state(DEV_IDLE)`.

## Data flow

Receipt ingest → `DEV_QR` → `ui_set_countdown(N)` → widget ticks `N→0` at 1 Hz
(label + bar) → at 0 sets expired flag → `poll_task` consumes it → `DEV_IDLE`.
Admin changes the timeout → `config-changed` → device re-pulls → the next QR uses
the new value.

## Error / edge handling

- Missing/invalid `qrTimeoutSeconds` → 60 s default (both admin normalize and
  device parse clamp).
- A tap during `DEV_QR` keeps its existing behavior (re-trigger/return) and
  should also stop the countdown (the next `ui_render_state` tears the timer down).
- A receipt screen with no countdown object → `ui_set_countdown` is a no-op
  (no widget registered); auto-return still fires off the expired flag, OR — to
  avoid auto-returning when there's no countdown shown — only arm the expiry when
  a countdown widget is actually present. **Chosen:** arm the deadline inside
  `render_countdown` (so no countdown object ⇒ no auto-return), and `ui_set_countdown`
  records the duration the next render should use.

## Testing

- **Host (`cfg-harness`):** `format_mmss` (0→`0:00`, 48→`0:48`, 125→`2:05`,
  600→`10:00`); `cfg_parse` default (absent→60) + clamp (5→15, 999→180).
- **Admin (vitest):** `normalizePrinterConfig` default + clamp for
  `qrTimeoutSeconds`.
- **Hardware:** receipt → QR shows, countdown ticks down, bar depletes, device
  auto-returns to idle at ~0:00; tap still returns immediately; changing the
  timeout in branding takes effect on the next receipt.

## Out of scope

- `pairingCode` / `steps` widgets (M6).
- Per-screen (vs per-tenant) timeout.
- Animating the countdown in the admin preview (kept static).
