# Downscale-on-upload for logos & icons — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Normalize every tenant branding image upload (logo + icon) to a device-decodable PNG bounded at 512px longest side, so oversized images and SVGs stop silently failing on the printer device.

**Architecture:** Add one focused helper `lib/image.ts#normalizeUploadImage` built on `sharp` (libvips) that decodes any supported input (PNG/JPEG/WebP/SVG), fits it within 512×512 preserving aspect (never upscaling), and re-encodes to PNG. Wire it into the two upload call sites in the branding server action. Register `sharp` as an external server package so Next doesn't bundle the native module.

**Tech Stack:** Next.js 16 server actions · `sharp` · vitest · R2 (`putObject`).

Spec: `docs/superpowers/specs/2026-06-17-logo-downscale-upload-design.md`

---

### Task 1: Add the `sharp` dependency and mark it external

**Files:**
- Modify: `package.json` (dependencies)
- Modify: `next.config.ts:6`

- [ ] **Step 1: Install sharp**

Run: `npm install sharp`
Expected: `sharp` appears under `dependencies` in `package.json`; install succeeds (native binary fetched for the local platform).

- [ ] **Step 2: Mark sharp external in next.config.ts**

Modify `next.config.ts` so the `serverExternalPackages` array includes `"sharp"`:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // Better Auth pulls in optional adapter dialects (e.g. kysely's bun:sqlite)
  // that must not be bundled — keep it external so Node resolves it at runtime.
  // sharp is a native (libvips) module and must likewise stay unbundled.
  serverExternalPackages: ["better-auth", "@better-auth/kysely-adapter", "sharp"],
};

export default nextConfig;
```

- [ ] **Step 3: Verify the app still builds**

Run: `npm run build`
Expected: build completes without errors (sharp resolves, no bundling error).

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json next.config.ts
git commit -m "build: add sharp dependency, mark external for server runtime"
```

---

### Task 2: `normalizeUploadImage` helper (TDD)

**Files:**
- Create: `lib/image.ts`
- Test: `lib/image.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `lib/image.test.ts`. The tests build their own fixtures with `sharp` so they need no binary assets. PNG dimensions are read straight from the IHDR (width at byte offset 16, height at 20, both big-endian).

```ts
import { describe, it, expect } from "vitest";
import sharp from "sharp";
import { normalizeUploadImage, MAX_IMAGE_DIM } from "./image";

const PNG_MAGIC = Buffer.from([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a]);

function pngDims(buf: Buffer) {
  return { w: buf.readUInt32BE(16), h: buf.readUInt32BE(20) };
}

// A solid-color raster of the given size, in the given format.
function raster(width: number, height: number, format: "png" | "jpeg") {
  return sharp({
    create: { width, height, channels: 3, background: { r: 10, g: 120, b: 200 } },
  })[format]().toBuffer();
}

describe("normalizeUploadImage", () => {
  it("downscales a large image so the longest side is <= MAX_IMAGE_DIM, preserving aspect and PNG output", async () => {
    const input = await raster(2000, 1000, "png");
    const out = await normalizeUploadImage(input);
    expect(out.subarray(0, 8).equals(PNG_MAGIC)).toBe(true);
    const { w, h } = pngDims(out);
    expect(Math.max(w, h)).toBeLessThanOrEqual(MAX_IMAGE_DIM);
    expect(w).toBe(512); // 2000x1000 fit into 512 box -> 512x256
    expect(h).toBe(256);
  });

  it("does not upscale an image already smaller than the cap", async () => {
    const input = await raster(100, 80, "png");
    const out = await normalizeUploadImage(input);
    const { w, h } = pngDims(out);
    expect(w).toBe(100);
    expect(h).toBe(80);
  });

  it("re-encodes a non-PNG raster (JPEG) to PNG", async () => {
    const input = await raster(300, 300, "jpeg");
    const out = await normalizeUploadImage(input);
    expect(out.subarray(0, 8).equals(PNG_MAGIC)).toBe(true);
  });

  it("rasterizes an SVG to a PNG", async () => {
    const svg = Buffer.from(
      `<svg xmlns="http://www.w3.org/2000/svg" width="800" height="400"><rect width="800" height="400" fill="#10A765"/></svg>`,
    );
    const out = await normalizeUploadImage(svg);
    expect(out.subarray(0, 8).equals(PNG_MAGIC)).toBe(true);
    const { w, h } = pngDims(out);
    expect(Math.max(w, h)).toBeLessThanOrEqual(MAX_IMAGE_DIM);
    expect(w).toBe(512); // 800x400 -> 512x256
  });

  it("throws on undecodable input", async () => {
    await expect(normalizeUploadImage(Buffer.from("not an image at all"))).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run lib/image.test.ts`
Expected: FAIL — `lib/image.ts` does not exist / `normalizeUploadImage` is not defined.

- [ ] **Step 3: Implement `lib/image.ts`**

```ts
import sharp from "sharp";

/** Longest-side ceiling for stored branding images (logo + icons). The printer
 *  device decodes PNGs to a width*height*4 ARGB bitmap in PSRAM, so native
 *  pixel size — not the on-screen box — drives memory use. 512px ≈ 1 MB decode,
 *  comfortably within the board's PSRAM and ample for the ~524px logo box. */
export const MAX_IMAGE_DIM = 512;

/**
 * Decode any supported image (PNG/JPEG/WebP/SVG), fit it within
 * MAX_IMAGE_DIM x MAX_IMAGE_DIM preserving aspect ratio, never upscale, and
 * re-encode to PNG so the printer device (lodepng, PNG-only) can always decode
 * it. Throws if the bytes cannot be decoded as an image.
 */
export async function normalizeUploadImage(bytes: Buffer): Promise<Buffer> {
  const probe = sharp(bytes);
  const meta = await probe.metadata();

  // SVG is vector: resizing after a default-density rasterization blurs. Render
  // it at a density that targets ~MAX_IMAGE_DIM on the long side up front.
  let pipeline = probe;
  if (meta.format === "svg") {
    const longSide = Math.max(meta.width ?? MAX_IMAGE_DIM, meta.height ?? MAX_IMAGE_DIM);
    const density = Math.min(2400, Math.max(72, Math.round((72 * MAX_IMAGE_DIM) / longSide)));
    pipeline = sharp(bytes, { density });
  }

  return pipeline
    .resize(MAX_IMAGE_DIM, MAX_IMAGE_DIM, { fit: "inside", withoutEnlargement: true })
    .png()
    .toBuffer();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run lib/image.test.ts`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add lib/image.ts lib/image.test.ts
git commit -m "feat: normalizeUploadImage — fit branding uploads to 512px PNG"
```

---

### Task 3: Wire `normalizeUploadImage` into the logo upload

**Files:**
- Modify: `app/(tenant)/tenant/branding/actions.ts:17` (import) and `:120-135` (logo block)

- [ ] **Step 1: Add the import**

In `app/(tenant)/tenant/branding/actions.ts`, add below the existing storage import (line 17):

```ts
import { normalizeUploadImage } from "@/lib/image";
```

- [ ] **Step 2: Normalize logo bytes before upload**

Replace the logo upload block (currently lines ~120-135):

```ts
  if (logo instanceof File && logo.size > 0) {
    if (!logo.type.startsWith("image/")) {
      return { ok: false, error: "Logo must be an image file." };
    }
    if (logo.size > MAX_LOGO_BYTES) {
      return { ok: false, error: "Logo must be under 2 MB." };
    }
    const key = logoStorageKey(organizationId, id("logo"));
    const bytes = Buffer.from(await logo.arrayBuffer());
    try {
      await putObject(key, bytes, logo.type);
    } catch (err) {
      console.error("Logo upload failed", err);
      return { ok: false, error: "Logo upload failed. Try again." };
    }
    logoUrlUpdate = key;
  } else if (removeLogo) {
```

with:

```ts
  if (logo instanceof File && logo.size > 0) {
    if (!logo.type.startsWith("image/")) {
      return { ok: false, error: "Logo must be an image file." };
    }
    if (logo.size > MAX_LOGO_BYTES) {
      return { ok: false, error: "Logo must be under 2 MB." };
    }
    let bytes: Buffer;
    try {
      bytes = await normalizeUploadImage(Buffer.from(await logo.arrayBuffer()));
    } catch {
      return { ok: false, error: "Couldn't process that image — try a PNG or JPEG." };
    }
    const key = logoStorageKey(organizationId, id("logo"));
    try {
      await putObject(key, bytes, "image/png");
    } catch (err) {
      console.error("Logo upload failed", err);
      return { ok: false, error: "Logo upload failed. Try again." };
    }
    logoUrlUpdate = key;
  } else if (removeLogo) {
```

- [ ] **Step 3: Verify typecheck passes**

Run: `npx tsc --noEmit`
Expected: no new errors in `app/(tenant)/tenant/branding/actions.ts`.

- [ ] **Step 4: Commit**

```bash
git add "app/(tenant)/tenant/branding/actions.ts"
git commit -m "feat: normalize logo uploads to 512px PNG before R2 store"
```

---

### Task 4: Wire `normalizeUploadImage` into the icon upload

**Files:**
- Modify: `app/(tenant)/tenant/branding/actions.ts:74-90` (icon block)

- [ ] **Step 1: Normalize icon bytes before upload**

Replace the icon upload block (currently lines ~74-90):

```ts
          if (file instanceof File && file.size > 0) {
            if (!file.type.startsWith("image/")) {
              return { ok: false, error: "Icon must be an image file." };
            }
            if (file.size > MAX_LOGO_BYTES) {
              return { ok: false, error: "Icon must be under 2 MB." };
            }
            const key = iconStorageKey(organizationId, id("icon"));
            const bytes = Buffer.from(await file.arrayBuffer());
            try {
              await putObject(key, bytes, file.type);
              o.icon = { ...o.icon, url: key };
            } catch (err) {
              console.error("Icon upload failed", err);
              return { ok: false, error: "Icon upload failed. Try again." };
            }
          } else {
```

with:

```ts
          if (file instanceof File && file.size > 0) {
            if (!file.type.startsWith("image/")) {
              return { ok: false, error: "Icon must be an image file." };
            }
            if (file.size > MAX_LOGO_BYTES) {
              return { ok: false, error: "Icon must be under 2 MB." };
            }
            let bytes: Buffer;
            try {
              bytes = await normalizeUploadImage(Buffer.from(await file.arrayBuffer()));
            } catch {
              return { ok: false, error: "Couldn't process that icon — try a PNG or JPEG." };
            }
            const key = iconStorageKey(organizationId, id("icon"));
            try {
              await putObject(key, bytes, "image/png");
              o.icon = { ...o.icon, url: key };
            } catch (err) {
              console.error("Icon upload failed", err);
              return { ok: false, error: "Icon upload failed. Try again." };
            }
          } else {
```

- [ ] **Step 2: Verify typecheck passes**

Run: `npx tsc --noEmit`
Expected: no new errors in `app/(tenant)/tenant/branding/actions.ts`.

- [ ] **Step 3: Run the full test suite**

Run: `npm run test`
Expected: all tests pass, including the new `lib/image.test.ts`.

- [ ] **Step 4: Commit**

```bash
git add "app/(tenant)/tenant/branding/actions.ts"
git commit -m "feat: normalize icon uploads to 512px PNG before R2 store"
```

---

### Task 5: Manual end-to-end verification (no code)

**Files:** none.

- [ ] **Step 1: Start the dev server**

Run: `npm run dev`

- [ ] **Step 2: Upload an oversized image**

In the tenant branding editor, upload a >2000px PNG as the logo and save. Confirm the action succeeds and the stored logo (inspect via the device-config / R2) is ≤512px PNG.

- [ ] **Step 3: Upload an SVG**

Upload an `.svg` logo and save. Confirm it succeeds and the stored asset is a PNG (previously this stored an unrenderable SVG).

- [ ] **Step 4: Confirm on device (optional, if hardware present)**

Save triggers `config-changed`; within ~12s the printer device prefetches and renders the normalized logo. Confirm it appears (do not open serial — it reboots the board).

---

## Self-Review

**Spec coverage:**
- Rasterize SVG → PNG — Task 2 (SVG branch + test) ✓
- Cover logos + icons — Tasks 3 & 4 ✓
- Uniform 512px cap — `MAX_IMAGE_DIM = 512`, Task 2 ✓
- Normalize all to PNG — `.png()` + `putObject(…, "image/png")`, Tasks 2-4 ✓
- `sharp` + `serverExternalPackages` — Task 1 ✓
- Friendly error on undecodable input — Tasks 3 & 4 ✓
- Tests (large/small/SVG/garbage) — Task 2 ✓ (small-raster covers "no upscale"; large covers downscale+aspect; plus a JPEG→PNG case)

**Placeholder scan:** none — every code step shows full code; every command has expected output.

**Type consistency:** `normalizeUploadImage(bytes: Buffer): Promise<Buffer>` and `MAX_IMAGE_DIM` are used identically across Tasks 2-4. Both call sites store with literal `"image/png"`. `id`, `logoStorageKey`, `iconStorageKey`, `putObject`, `MAX_LOGO_BYTES` already exist in `actions.ts`.
