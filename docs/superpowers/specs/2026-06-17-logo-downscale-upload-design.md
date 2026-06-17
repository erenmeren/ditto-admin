# Downscale-on-upload for logos & icons

**Date:** 2026-06-17
**Status:** Approved (design)
**Branch:** `feat/logo-downscale-upload`

## Problem

The tenant branding editor accepts logo and icon image uploads with only two
checks: MIME type `image/*` and file size < 2 MB (`MAX_LOGO_BYTES`). The raw
bytes are stored to R2 as-is and served to the printer device via a presigned
URL. The device (ESP32-P4, LVGL + lodepng) decodes the PNG into an
uncompressed ARGB8888 bitmap of `width × height × 4` bytes in PSRAM.

Two real failures result, both **silent on the device** (firmware
`CONFIG_LV_USE_LOG` is off, so a decode failure produces no log and a blank
logo area):

1. **Oversized images.** A user uploaded a 6000×6000 PNG (402 KB encoded, under
   the 2 MB gate). Decoded = 6000·6000·4 = **144 MB**, far past the board's
   ~32 MB PSRAM → allocation fails → nothing renders.
2. **SVG.** `image/svg+xml` passes the `startsWith("image/")` check, but the
   device has no vector renderer (lodepng is PNG-only) → SVG never renders.

The admin gives the user zero feedback in either case. This spec closes the gap
by normalizing every upload to a device-decodable PNG at a bounded size.

## Decisions

- **Rasterize SVG → PNG** on upload (accept SVGs, don't reject them).
- **Cover both logos and icons** — they share the identical device decode path.
- **Uniform 512px** longest-side cap (one constant for both).
- **Normalize all uploads to PNG** regardless of input format.

512px matches the widest on-device logo box (~524px on the 720px panel) and
decodes to ~1 MB ARGB — comfortably within PSRAM with margin for several
prefetched icons.

## Approach

Use **`sharp`** (libvips) — the standard image library for Next.js/Vercel. It
decodes PNG/JPEG/WebP/SVG and re-encodes to PNG in a single pass. The
alternative (pure-JS `pngjs` + hand-rolled resize + a separate SVG rasterizer)
is more code, slower, and has poor SVG support.

Trade-off: `sharp` is a native module, so it must be added to
`serverExternalPackages` in `next.config.ts` (the same mechanism already used
for `better-auth`) so Next does not attempt to bundle it.

## Components

### 1. `lib/image.ts` (new) — single focused helper

```ts
export const MAX_IMAGE_DIM = 512;

/**
 * Decode any supported image (PNG/JPEG/WebP/SVG), fit within
 * MAX_IMAGE_DIM × MAX_IMAGE_DIM preserving aspect ratio, never upscale, and
 * re-encode to PNG so the printer device can always decode it.
 * Throws if the input cannot be decoded.
 */
export async function normalizeUploadImage(bytes: Buffer): Promise<Buffer>;
```

Implementation notes:
- `sharp(bytes).resize(MAX_IMAGE_DIM, MAX_IMAGE_DIM, { fit: "inside",
  withoutEnlargement: true }).png().toBuffer()`.
  - `fit: "inside"` preserves aspect, fitting within the 512×512 box.
  - `withoutEnlargement: true` never upscales images already ≤512px.
  - `.png()` re-encodes (JPEG/WebP/SVG → PNG; normalizes color type).
- **SVG crispness:** rasterizing an SVG then resizing can blur. Render SVG
  inputs at a density that yields a ~512px raster directly. Detect SVG via
  `sharp(bytes).metadata()` (`format === "svg"`) and re-open with an
  appropriate `density` (e.g. scale `density` so the rendered long side ≈ 512)
  before the resize. Raster inputs skip this.
- **Decompression-bomb safety:** rely on sharp's built-in `limitInputPixels`
  (default ~268 Mpx; 6000×6000 = 36 Mpx is within it). The existing 2 MB
  file-size gate remains the first line of defense.

### 2. `app/(tenant)/tenant/branding/actions.ts` — two call sites

Both the logo block (~L120-138) and the icon block (~L74-90) currently do:
MIME check → size check → `putObject(key, rawBytes, file.type)`.

Change: after the existing MIME + size checks, pass the bytes through
`normalizeUploadImage` and store the result as PNG:

```ts
let normalized: Buffer;
try {
  normalized = await normalizeUploadImage(Buffer.from(await file.arrayBuffer()));
} catch {
  return { ok: false, error: "Couldn't process that image — try a PNG or JPEG." };
}
await putObject(key, normalized, "image/png");
```

The `startsWith("image/")` MIME gate stays (rejects non-image uploads early).
SVG now flows through and is rasterized rather than stored unrenderable.

### 3. `next.config.ts`

Add `"sharp"` to the existing `serverExternalPackages` array.

## Data flow

upload → MIME gate → 2 MB gate → `normalizeUploadImage` (decode → fit 512 →
PNG) → `putObject(…, "image/png")` → existing `logoUrl` / icon-key persistence
+ `config-changed` enqueue (all downstream unchanged) → device prefetch →
lodepng decode (now always small + PNG) → render.

## Error handling

- Undecodable input → `normalizeUploadImage` throws → action returns a friendly
  `{ ok: false, error }`, no R2 write, no DB change.
- Existing previous-asset cleanup logic is unaffected (still keyed on the
  stored object key).

## Testing

`lib/image.test.ts` (vitest):
- Large 1000×1000 PNG → output longest side ≤ 512, PNG magic bytes, aspect
  ratio preserved.
- Small 100×100 PNG → returned unchanged in dimensions (no upscale).
- Minimal valid SVG → output is a PNG.
- Garbage / non-image bytes → throws.

## Out of scope

- Client-side preview resizing (server is the trust boundary).
- Changing the 2 MB upload gate.
- Backfilling existing stored logos (the one oversized asset we hit is already
  fixed manually).
- Per-type size caps (uniform 512 chosen for simplicity).
