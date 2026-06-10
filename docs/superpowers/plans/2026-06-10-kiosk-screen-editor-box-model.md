# Kiosk Screen Editor Box-Model Rework — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the kiosk idle-screen editor on a real box model (objects with pixel width/height, top-left anchor) so dragging has no jump, resizing changes a real box without distorting content, all text is editable with a font-size control, and objects snap/align like SquareLine.

**Architecture:** Replace `sx/sy` scale-transform with `KioskObject {type,x,y,w,h,...}` boxes (fractions of the 720 canvas). Pure geometry (`resizeBox` on top-left boxes + `snapMove`/`snapResize`) and the data model (`normalizeKioskLayout` v2) are rewritten test-first. Rendering positions each object as a real box; text wraps inside with an independent font size; logo/clock/wifi size deterministically from their box (no DOM measurement, no scale transform). The editor hook gains grab-offset drag, box resize, and snap guides; the controls become a type-aware properties panel.

**Tech Stack:** Next.js 16 / React 19 / TypeScript (strict), Tailwind v4 + shadcn (radix-nova), lucide-react, Vitest. No new deps, no DB migration.

---

## File structure

- **Rewrite** `lib/kiosk-geometry.ts` (+ test) — top-left `Box`, `resizeBox`, `snapMove`, `snapResize`, `Guides`, `HANDLES`, `MIN_BOX`.
- **Rewrite** `lib/kiosk-layout.ts` (+ test) — `KioskObject`/`KioskObjectType`, `KioskLayout {version:2, objects}`, `normalizeKioskLayout` (v1→default reset), `createTextObject`, `objectLabel`, `defaultLayout()`/`DEFAULT_KIOSK_LAYOUT`, `OBJECT_TYPES`/`FIXED_TYPES`/`TYPE_LABEL`, `FONT_MIN/MAX`, `MAX_CUSTOM`, `MAX_TEXT_LEN`.
- **Rewrite** `components/device-preview/kiosk-preview.tsx` idle rendering — `IdleScreen` maps objects to positioned boxes; new `ObjectVisual`/`TextObject`/`LogoObject`/`ClockObject`/`WifiObject`; `KioskClock` gains a `size` prop. Other 6 screens unchanged.
- **Rewrite** `components/device-preview/kiosk-editor/use-kiosk-editor.ts` — box model: grab-offset move, box resize, snapping, `guides`, derived `selBox`, object handlers; drop `elRefs`/measurement.
- **Rewrite** `components/device-preview/kiosk-editor/kiosk-stage.tsx` — render object boxes; selection overlay + 8 handles on the selected box; snap guide lines.
- **Rewrite** `components/device-preview/kiosk-editor/kiosk-controls.tsx` — object list + type-aware properties panel (text/font/align, clock tz/24h, wifi level, X/Y/W/H).
- **Edit** `components/branding-editor.tsx` — drop the now-unused `remeasureKey` hook arg; everything else (shell, save bar, Brand/Security) unchanged.

**Build note:** the model change ripples through every file at once, so intermediate `tsc` won't be green until Task 5. Each task keeps **Vitest** green (`npx vitest run`); the first full `npx tsc --noEmit` + `npm run build` is Task 5/6.

---

## Task 1: Geometry — top-left box + snapping (`lib/kiosk-geometry.ts`)

**Files:**
- Rewrite: `lib/kiosk-geometry.ts`
- Rewrite: `lib/kiosk-geometry.test.ts`

- [ ] **Step 1: Replace the test file**

```ts
import { describe, it, expect } from "vitest";
import { resizeBox, snapMove, snapResize, clampToCanvas, MIN_BOX, type Box } from "./kiosk-geometry";

const box: Box = { x: 0.4, y: 0.45, w: 0.2, h: 0.1 }; // edges L0.4 R0.6 T0.45 B0.55

describe("resizeBox (top-left box)", () => {
  it("east edge changes width, left fixed", () => {
    const r = resizeBox(box, "e", { x: 0.8, y: 0.99 });
    expect(r.x).toBeCloseTo(0.4, 6);
    expect(r.w).toBeCloseTo(0.4, 6);
    expect(r.h).toBeCloseTo(0.1, 6);
  });
  it("west edge changes x and width, right fixed", () => {
    const r = resizeBox(box, "w", { x: 0.5, y: 0.5 });
    expect(r.x).toBeCloseTo(0.5, 6);
    expect(r.w).toBeCloseTo(0.1, 6); // R0.6 - 0.5
  });
  it("south-east corner changes width and height independently", () => {
    const r = resizeBox(box, "se", { x: 0.9, y: 0.85 });
    expect(r.w).toBeCloseTo(0.5, 6);
    expect(r.h).toBeCloseTo(0.4, 6);
  });
  it("floors at MIN_BOX and never inverts", () => {
    const r = resizeBox(box, "e", { x: 0.1, y: 0.5 }); // dragged past left
    expect(r.w).toBeGreaterThanOrEqual(MIN_BOX);
    expect(r.x).toBeCloseTo(0.4, 6);
  });
  it("north edge dragged past the bottom floors height and re-anchors top", () => {
    const r = resizeBox(box, "n", { x: 0.5, y: 0.9 }); // bottom is 0.55
    expect(r.h).toBe(MIN_BOX);
    expect(r.y).toBeCloseTo(0.55 - MIN_BOX, 6); // re-anchored to bottom - MIN_BOX
  });
  it("nw corner dragged past se floors BOTH axes and re-anchors to the se corner", () => {
    const r = resizeBox(box, "nw", { x: 0.99, y: 0.99 }); // right 0.6, bottom 0.55
    expect(r.w).toBe(MIN_BOX);
    expect(r.h).toBe(MIN_BOX);
    expect(r.x).toBeCloseTo(0.6 - MIN_BOX, 6);
    expect(r.y).toBeCloseTo(0.55 - MIN_BOX, 6);
  });
});

describe("snapMove", () => {
  it("snaps the box center to the canvas center", () => {
    const moving: Box = { x: 0.39, y: 0.45, w: 0.2, h: 0.1 }; // centerX 0.49, near 0.5
    const { box: r, guides } = snapMove(moving, [], 0.02);
    expect(r.x + r.w / 2).toBeCloseTo(0.5, 6);
    expect(guides.vx).toContain(0.5);
  });
  it("snaps the left edge to another object's left edge", () => {
    const other: Box = { x: 0.2, y: 0.0, w: 0.1, h: 0.1 };
    const moving: Box = { x: 0.205, y: 0.5, w: 0.1, h: 0.1 };
    const { box: r, guides } = snapMove(moving, [other], 0.02);
    expect(r.x).toBeCloseTo(0.2, 6);
    expect(guides.vx).toContain(0.2);
  });
  it("does not snap outside the threshold", () => {
    const moving: Box = { x: 0.1, y: 0.1, w: 0.2, h: 0.1 };
    const { box: r, guides } = snapMove(moving, [], 0.005);
    expect(r.x).toBeCloseTo(0.1, 6);
    expect(guides.vx).toHaveLength(0);
    expect(guides.hy).toHaveLength(0);
  });
  it("snaps the top edge to the canvas top and reports a horizontal guide", () => {
    const moving: Box = { x: 0.5, y: 0.01, w: 0.1, h: 0.1 }; // top near 0
    const { box: r, guides } = snapMove(moving, [], 0.02);
    expect(r.y).toBeCloseTo(0, 6);
    expect(guides.hy).toContain(0);
  });
});

describe("snapResize", () => {
  it("snaps the dragged east edge to the canvas right and reports a guide", () => {
    const b: Box = { x: 0.2, y: 0.2, w: 0.78, h: 0.2 }; // right edge 0.98, near 1
    const { box: r, guides } = snapResize(b, "e", [], 0.03);
    expect(r.x + r.w).toBeCloseTo(1, 6);
    expect(guides.vx).toContain(1);
  });
  it("leaves the non-dragged edges alone", () => {
    const b: Box = { x: 0.2, y: 0.2, w: 0.6, h: 0.6 };
    const { box: r } = snapResize(b, "e", [], 0.03);
    expect(r.x).toBeCloseTo(0.2, 6);
    expect(r.y).toBeCloseTo(0.2, 6);
    expect(r.h).toBeCloseTo(0.6, 6);
  });
  it("snaps the south edge to the canvas bottom", () => {
    const b: Box = { x: 0.2, y: 0.2, w: 0.2, h: 0.78 }; // bottom 0.98, near 1
    const { box: r, guides } = snapResize(b, "s", [], 0.03);
    expect(r.y + r.h).toBeCloseTo(1, 6);
    expect(guides.hy).toContain(1);
  });
  it("does NOT report a guide when MIN_BOX overrides the snap", () => {
    // east drag on a tiny box: right 0.53 snaps toward 0.5, but that collapses
    // width below MIN_BOX, so the edge is pushed back out — no honest guide.
    const b: Box = { x: 0.49, y: 0.2, w: 0.04, h: 0.2 };
    const { box: r, guides } = snapResize(b, "e", [], 0.04);
    expect(r.w).toBeGreaterThanOrEqual(MIN_BOX);
    expect(guides.vx).toHaveLength(0);
  });
});

describe("clampToCanvas", () => {
  it("pulls a box back onto the canvas", () => {
    expect(clampToCanvas({ x: 0.9, y: 0.95, w: 0.3, h: 0.2 })).toMatchObject({ x: 0.7, y: 0.8 });
  });
  it("clamps an oversized box and floors a tiny one", () => {
    const big = clampToCanvas({ x: -1, y: -1, w: 5, h: 5 });
    expect(big).toMatchObject({ x: 0, y: 0, w: 1, h: 1 });
    expect(clampToCanvas({ x: 0.5, y: 0.5, w: 0, h: 0 }).w).toBe(MIN_BOX);
  });
});
```

- [ ] **Step 2: Run the tests — expect FAIL**

Run: `npx vitest run lib/kiosk-geometry.test.ts`
Expected: FAIL — new exports/shape not present.

- [ ] **Step 3: Replace `lib/kiosk-geometry.ts`**

```ts
// Pure resize + snap geometry for the kiosk editor. Operates on a top-left box in
// canvas fractions (0..1); no React/DOM so it is unit-testable.

export interface Box {
  x: number; // left, fraction 0..1
  y: number; // top, fraction 0..1
  w: number; // width, fraction 0..1
  h: number; // height, fraction 0..1
}

/** The 8 resize handles. n/s/e/w = edges; nw/ne/sw/se = corners. */
export const HANDLES = ["nw", "n", "ne", "e", "se", "s", "sw", "w"] as const;
export type Handle = (typeof HANDLES)[number];

/** Active alignment guide lines to draw: vertical at x's, horizontal at y's. */
export interface Guides {
  vx: number[];
  hy: number[];
}

/** Minimum box size as a fraction of the canvas — keeps objects grabbable. */
export const MIN_BOX = 0.04;

/** Distance tie-break epsilon for snapping (ignore sub-nanometer float noise). */
const SNAP_EPS = 1e-9;

const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));

/**
 * Resize `box` by dragging `handle` to `pointer` (canvas fractions). The edge or
 * corner opposite the handle stays fixed; width/height are floored at MIN_BOX and
 * never invert.
 */
export function resizeBox(box: Box, handle: Handle, pointer: { x: number; y: number }): Box {
  let left = box.x;
  let top = box.y;
  let right = box.x + box.w;
  let bottom = box.y + box.h;

  const movesE = handle.includes("e");
  const movesW = handle.includes("w");
  const movesS = handle.includes("s");
  const movesN = handle.includes("n");

  if (movesE) right = pointer.x;
  if (movesW) left = pointer.x;
  if (movesS) bottom = pointer.y;
  if (movesN) top = pointer.y;

  // Enforce the MIN_BOX floor by setting the dimension exactly and re-anchoring
  // the moving edge — avoids floating-point underflow from re-subtraction
  // (e.g. (0.4 + 0.04) - 0.4 = 0.03999…).
  let w = right - left;
  let h = bottom - top;
  if (w < MIN_BOX) { w = MIN_BOX; if (movesW) left = right - MIN_BOX; else right = left + MIN_BOX; }
  if (h < MIN_BOX) { h = MIN_BOX; if (movesN) top = bottom - MIN_BOX; else bottom = top + MIN_BOX; }

  return { x: left, y: top, w, h };
}

/** Canvas + other-object snap targets along each axis. */
function targetsX(others: Box[]): number[] {
  const t = [0, 0.5, 1];
  for (const o of others) t.push(o.x, o.x + o.w / 2, o.x + o.w);
  return t;
}
function targetsY(others: Box[]): number[] {
  const t = [0, 0.5, 1];
  for (const o of others) t.push(o.y, o.y + o.h / 2, o.y + o.h);
  return t;
}

/** Best snap of any probe to any target within threshold → {delta, line} or null. */
function bestSnap(probes: number[], targets: number[], threshold: number): { delta: number; line: number } | null {
  let best: { delta: number; line: number; dist: number } | null = null;
  for (const p of probes) {
    for (const t of targets) {
      const dist = Math.abs(t - p);
      // SNAP_EPS tie-break: keep the earlier probe (left/top before center before
      // right/bottom) on near-equal distances, so floating-point noise can't flip it.
      if (dist <= threshold && (!best || dist < best.dist - SNAP_EPS)) best = { delta: t - p, line: t, dist };
    }
  }
  return best ? { delta: best.delta, line: best.line } : null;
}

/** Snap nearest target to a single value (an edge) within threshold, or null. */
function snapValue(value: number, targets: number[], threshold: number): number | null {
  let best: { v: number; dist: number } | null = null;
  for (const t of targets) {
    const dist = Math.abs(t - value);
    if (dist <= threshold && (!best || dist < best.dist - SNAP_EPS)) best = { v: t, dist };
  }
  return best ? best.v : null;
}

/**
 * Snap a moving box (left/center/right & top/middle/bottom probes) to the canvas
 * lines (0/0.5/1) and other objects' edges/centers. Returns the snapped box and
 * the guide lines that became active.
 */
export function snapMove(box: Box, others: Box[], threshold: number): { box: Box; guides: Guides } {
  const sx = bestSnap([box.x, box.x + box.w / 2, box.x + box.w], targetsX(others), threshold);
  const sy = bestSnap([box.y, box.y + box.h / 2, box.y + box.h], targetsY(others), threshold);
  return {
    box: { ...box, x: box.x + (sx?.delta ?? 0), y: box.y + (sy?.delta ?? 0) },
    guides: { vx: sx ? [sx.line] : [], hy: sy ? [sy.line] : [] },
  };
}

/**
 * Snap the dragged edges of a (already-resized) box to the canvas lines and other
 * objects' edges, keeping the opposite edges fixed and the MIN_BOX floor.
 */
export function snapResize(box: Box, handle: Handle, others: Box[], threshold: number): { box: Box; guides: Guides } {
  let left = box.x;
  let top = box.y;
  let right = box.x + box.w;
  let bottom = box.y + box.h;
  const tx = targetsX(others);
  const ty = targetsY(others);

  // Snap the dragged edges...
  const sE = handle.includes("e") ? snapValue(right, tx, threshold) : null;
  const sW = handle.includes("w") ? snapValue(left, tx, threshold) : null;
  const sS = handle.includes("s") ? snapValue(bottom, ty, threshold) : null;
  const sN = handle.includes("n") ? snapValue(top, ty, threshold) : null;
  if (sE != null) right = sE;
  if (sW != null) left = sW;
  if (sS != null) bottom = sS;
  if (sN != null) top = sN;

  // ...then enforce the MIN_BOX floor (which may override a snap).
  if (right - left < MIN_BOX) { if (handle.includes("w")) left = right - MIN_BOX; else right = left + MIN_BOX; }
  if (bottom - top < MIN_BOX) { if (handle.includes("n")) top = bottom - MIN_BOX; else bottom = top + MIN_BOX; }

  // Emit a guide only for snaps that actually held after MIN_BOX recovery, so a
  // guide line never lies about where an edge sits.
  const vx: number[] = [];
  const hy: number[] = [];
  if (sE != null && Math.abs(right - sE) < SNAP_EPS) vx.push(sE);
  if (sW != null && Math.abs(left - sW) < SNAP_EPS) vx.push(sW);
  if (sS != null && Math.abs(bottom - sS) < SNAP_EPS) hy.push(sS);
  if (sN != null && Math.abs(top - sN) < SNAP_EPS) hy.push(sN);

  return { box: { x: left, y: top, w: right - left, h: bottom - top }, guides: { vx, hy } };
}

/** Clamp a box fully onto the canvas (size preserved where possible). */
export function clampToCanvas(box: Box): Box {
  const w = clamp(box.w, MIN_BOX, 1);
  const h = clamp(box.h, MIN_BOX, 1);
  return { x: clamp(box.x, 0, 1 - w), y: clamp(box.y, 0, 1 - h), w, h };
}
```

- [ ] **Step 4: Run the tests — expect PASS**

Run: `npx vitest run lib/kiosk-geometry.test.ts`
Expected: PASS (all blocks).

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-geometry.ts lib/kiosk-geometry.test.ts
git commit -m "feat(kiosk): top-left box geometry with resize + snap helpers"
```

---

## Task 2: Data model — v2 object layout (`lib/kiosk-layout.ts`)

**Files:**
- Rewrite: `lib/kiosk-layout.ts`
- Rewrite: `lib/kiosk-layout.test.ts`

- [ ] **Step 1: Replace the test file**

```ts
import { describe, it, expect } from "vitest";
import {
  normalizeKioskLayout,
  createTextObject,
  objectLabel,
  defaultLayout,
  DEFAULT_KIOSK_LAYOUT,
  FIXED_TYPES,
  FONT_MIN,
  FONT_MAX,
  MAX_CUSTOM,
} from "./kiosk-layout";

describe("normalizeKioskLayout", () => {
  it("returns the default layout for null/garbage/v1", () => {
    expect(normalizeKioskLayout(null)).toEqual(DEFAULT_KIOSK_LAYOUT);
    expect(normalizeKioskLayout("nope")).toEqual(DEFAULT_KIOSK_LAYOUT);
    expect(normalizeKioskLayout({})).toEqual(DEFAULT_KIOSK_LAYOUT);
    // a v1 layout (elements + sx/sy, no version:2) → reset to default
    expect(normalizeKioskLayout({ elements: [{ id: "logo", x: 0.5, y: 0.4, sx: 1, sy: 1 }] })).toEqual(DEFAULT_KIOSK_LAYOUT);
  });

  it("keeps a valid v2 layout and round-trips it", () => {
    const l = defaultLayout();
    expect(normalizeKioskLayout(l)).toEqual(l);
  });

  it("ensures exactly one of each fixed widget", () => {
    const l = normalizeKioskLayout({ version: 2, objects: [{ type: "logo", x: 0.1, y: 0.1, w: 0.2, h: 0.2 }] });
    for (const t of FIXED_TYPES) {
      expect(l.objects.filter((o) => o.type === t)).toHaveLength(1);
    }
  });

  it("clamps box coords, sizes, and font", () => {
    const l = normalizeKioskLayout({
      version: 2,
      objects: [{ type: "text", text: "Hi", x: -1, y: 9, w: 0, h: 5, fontSize: 9999 }],
    });
    const t = l.objects.find((o) => o.type === "text")!;
    expect(t.x).toBeGreaterThanOrEqual(0);
    expect(t.y).toBeLessThanOrEqual(1);
    expect(t.w).toBeGreaterThan(0);
    expect(t.w).toBeLessThanOrEqual(1);
    expect(t.fontSize).toBe(FONT_MAX);
  });

  it("drops text-less / unknown-type objects and caps text objects", () => {
    const objs = [
      { type: "text" }, // no text → dropped
      { type: "bogus", x: 0.1, y: 0.1, w: 0.1, h: 0.1 }, // unknown → dropped
      ...Array.from({ length: 30 }, (_, i) => ({ type: "text", text: `t${i}` })),
    ];
    const l = normalizeKioskLayout({ version: 2, objects: objs });
    expect(l.objects.filter((o) => o.type === "text")).toHaveLength(MAX_CUSTOM);
  });

  it("trims long text to 80 chars", () => {
    const l = normalizeKioskLayout({ version: 2, objects: [{ type: "text", text: "x".repeat(200) }] });
    expect(l.objects.find((o) => o.type === "text")!.text!.length).toBe(80);
  });

  it("validates timezone and clamps wifi", () => {
    expect(normalizeKioskLayout({ version: 2, objects: [], clockTimezone: "Mars/Phobos" }).clockTimezone).toBe("UTC");
    expect(normalizeKioskLayout({ version: 2, objects: [], wifiLevel: 9 }).wifiLevel).toBe(4);
  });
});

describe("createTextObject", () => {
  it("makes a centered text object with a unique text-* id", () => {
    const a = createTextObject("Hi", 5);
    expect(a.type).toBe("text");
    expect(a.id).toMatch(/^text-/);
    expect(a).toMatchObject({ text: "Hi", visible: true, z: 5, align: "center" });
    expect(createTextObject("Hi", 5).id).not.toBe(a.id);
  });
});

describe("objectLabel", () => {
  it("uses the type label for fixed widgets and truncated text for text", () => {
    const l = defaultLayout();
    expect(objectLabel(l.objects.find((o) => o.type === "clock")!)).toBe("Clock");
    expect(objectLabel(createTextObject("A very long custom label here", 0))).toContain("A very long");
  });
});
```

- [ ] **Step 2: Run the tests — expect FAIL**

Run: `npx vitest run lib/kiosk-layout.test.ts`
Expected: FAIL — new exports/shape not present.

- [ ] **Step 3: Replace `lib/kiosk-layout.ts`**

```ts
// Kiosk idle-screen layout (v2): a list of objects, each a real box (top-left
// x/y + w/h as fractions of the 720² canvas, resolution-independent). Object
// types: text (editable content + font size + align), and the fixed widgets
// logo/clock/wifi (one each, hideable, not deletable). Persisted per tenant as
// jsonb; always loaded through normalizeKioskLayout so malformed/old (v1) data
// can never break the render — v1 layouts are reset to the default.
import { isValidTimezone } from "./timezones";
import { MIN_BOX } from "./kiosk-geometry";

export const OBJECT_TYPES = ["text", "logo", "clock", "wifi"] as const;
export type KioskObjectType = (typeof OBJECT_TYPES)[number];

export const FIXED_TYPES = ["logo", "clock", "wifi"] as const;
export type FixedType = (typeof FIXED_TYPES)[number];

export const TYPE_LABEL: Record<KioskObjectType, string> = {
  text: "Text",
  logo: "Logo",
  clock: "Clock",
  wifi: "Wi-Fi signal",
};

export type TextAlign = "left" | "center" | "right";

export interface KioskObject {
  id: string;
  type: KioskObjectType;
  x: number; // top-left, fraction 0..1
  y: number;
  w: number; // size, fraction 0..1
  h: number;
  visible: boolean;
  z: number;
  text?: string;
  fontSize?: number; // px on the 720 reference
  align?: TextAlign;
}

export interface KioskLayout {
  version: 2;
  clockTimezone: string;
  clock24h: boolean;
  wifiLevel: number; // 0..4
  objects: KioskObject[];
}

export const FONT_MIN = 8;
export const FONT_MAX = 160;
export const MAX_CUSTOM = 20;
export const MAX_TEXT_LEN = 80;

const ALIGNS: TextAlign[] = ["left", "center", "right"];
const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));
function num(v: unknown, fallback: number): number {
  return typeof v === "number" && Number.isFinite(v) ? v : fallback;
}

/** Default box + props for each fixed widget, reproducing today's arrangement. */
const FIXED_DEFAULTS: Record<FixedType, Pick<KioskObject, "x" | "y" | "w" | "h">> = {
  wifi: { x: 0.82, y: 0.04, w: 0.1, h: 0.06 },
  logo: { x: 0.3, y: 0.28, w: 0.4, h: 0.22 },
  clock: { x: 0.25, y: 0.52, w: 0.5, h: 0.18 },
};

/** Two seeded text objects (the old lane + tagline lines), now editable. */
function seededText(): KioskObject[] {
  return [
    { id: "text-lane", type: "text", x: 0.06, y: 0.05, w: 0.4, h: 0.06, visible: true, z: 3, text: "Lane 1", fontSize: 19, align: "left" },
    { id: "text-tagline", type: "text", x: 0.15, y: 0.88, w: 0.7, h: 0.08, visible: true, z: 4, text: "Tap your card or pay at the reader to begin", fontSize: 18, align: "center" },
  ];
}

/** A fresh default layout (new object each call so callers can't mutate it). */
export function defaultLayout(): KioskLayout {
  const fixed: KioskObject[] = FIXED_TYPES.map((type, i) => ({
    id: type,
    type,
    ...FIXED_DEFAULTS[type],
    visible: true,
    z: i,
  }));
  return {
    version: 2,
    clockTimezone: "UTC",
    clock24h: false,
    wifiLevel: 3,
    objects: [...fixed, ...seededText()],
  };
}

export const DEFAULT_KIOSK_LAYOUT: KioskLayout = defaultLayout();

const DEFAULT_FONT: Record<KioskObjectType, number> = { text: 24, logo: 24, clock: 24, wifi: 24 };

/** A fresh custom text object, centered, on top (`z`). */
export function createTextObject(text: string, z: number): KioskObject {
  const rand = typeof crypto !== "undefined" && crypto.randomUUID
    ? crypto.randomUUID().slice(0, 8)
    : Math.floor(Math.random() * 1e9).toString(36);
  return {
    id: `text-${rand}`,
    type: "text",
    x: 0.35,
    y: 0.45,
    w: 0.3,
    h: 0.1,
    visible: true,
    z,
    text: text.slice(0, MAX_TEXT_LEN),
    fontSize: 24,
    align: "center",
  };
}

/** Display name for the object list / inspector. */
export function objectLabel(o: KioskObject): string {
  if (o.type !== "text") return TYPE_LABEL[o.type];
  const t = (o.text ?? "").trim();
  return t ? (t.length > 18 ? `${t.slice(0, 18)}…` : t) : "Text";
}

/** Clamp a box onto the canvas with a minimum size. */
function sanitizeBox(o: Record<string, unknown>, d: Pick<KioskObject, "x" | "y" | "w" | "h">) {
  const w = clamp(num(o.w, d.w), MIN_BOX, 1);
  const h = clamp(num(o.h, d.h), MIN_BOX, 1);
  const x = clamp(num(o.x, d.x), 0, 1 - w);
  const y = clamp(num(o.y, d.y), 0, 1 - h);
  return { x, y, w, h };
}

/**
 * Coerce arbitrary stored data into a valid v2 KioskLayout. Non-v2 input (incl.
 * legacy v1 sx/sy layouts) is reset to the default. Guarantees one of each fixed
 * widget, ≤ MAX_CUSTOM valid text objects, clamped boxes/fonts, a known timezone,
 * and wifi 0..4. Never throws.
 */
export function normalizeKioskLayout(raw: unknown): KioskLayout {
  const r = raw as { version?: unknown; objects?: unknown; clockTimezone?: unknown; clock24h?: unknown; wifiLevel?: unknown } | null;
  if (!r || typeof r !== "object" || r.version !== 2 || !Array.isArray(r.objects)) {
    return defaultLayout();
  }
  const list = r.objects as Record<string, unknown>[];

  // 1) Fixed widgets — one of each, in default z order.
  const objects: KioskObject[] = FIXED_TYPES.map((type, i) => {
    const found = list.find((o) => o && o.type === type) ?? {};
    return {
      id: type,
      type,
      ...sanitizeBox(found, FIXED_DEFAULTS[type]),
      visible: typeof found.visible === "boolean" ? found.visible : true,
      z: typeof found.z === "number" && Number.isFinite(found.z) ? found.z : i,
    };
  });

  // 2) Text objects, capped.
  let zNext = FIXED_TYPES.length;
  let kept = 0;
  for (const o of list) {
    if (kept >= MAX_CUSTOM) break;
    if (!o || o.type !== "text" || typeof o.text !== "string" || o.text.trim() === "") continue;
    const align = ALIGNS.includes(o.align as TextAlign) ? (o.align as TextAlign) : "center";
    objects.push({
      id: typeof o.id === "string" && o.id ? o.id : `text-${kept}`,
      type: "text",
      ...sanitizeBox(o, { x: 0.35, y: 0.45, w: 0.3, h: 0.1 }),
      visible: typeof o.visible === "boolean" ? o.visible : true,
      z: typeof o.z === "number" && Number.isFinite(o.z) ? o.z : zNext++,
      text: o.text.slice(0, MAX_TEXT_LEN),
      fontSize: clamp(num(o.fontSize, DEFAULT_FONT.text), FONT_MIN, FONT_MAX),
      align,
    });
    kept++;
  }

  const tz = typeof r.clockTimezone === "string" && isValidTimezone(r.clockTimezone)
    ? r.clockTimezone
    : "UTC";

  return {
    version: 2,
    clockTimezone: tz,
    clock24h: typeof r.clock24h === "boolean" ? r.clock24h : false,
    wifiLevel: clamp(Math.round(num(r.wifiLevel, 3)), 0, 4),
    objects,
  };
}
```

- [ ] **Step 4: Run the tests — expect PASS**

Run: `npx vitest run lib/kiosk-layout.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-layout.ts lib/kiosk-layout.test.ts
git commit -m "feat(kiosk): v2 object layout model (boxes, text/font/align, v1 reset)"
```

---

## Task 3: Rendering — box-driven objects (`kiosk-preview.tsx`, `kiosk-clock.tsx`)

**Files:**
- Modify: `components/device-preview/kiosk-clock.tsx`
- Modify: `components/device-preview/kiosk-preview.tsx`

IMPORTANT: read both files first. Keep the top-of-file `cq` helper, `kioskRootStyle`, `KioskBrand`, the `Logo` component, and the 6 non-idle screen components UNCHANGED. Replace only the import of `@/lib/kiosk-layout` types, the `IdleScreen`, and the exported `KioskElementView` (renamed to `ObjectVisual` usage) — see steps.

- [ ] **Step 1: Give `KioskClock` a `size` prop**

In `components/device-preview/kiosk-clock.tsx`, change the signature and font sizing so the clock scales with a passed design-px `size` (the time font size). Replace the component's props + the two font sizes:

```tsx
export function KioskClock({
  timezone,
  hour24 = false,
  size = 84,
}: {
  timezone: string;
  hour24?: boolean;
  size?: number;
}) {
```
Then set the time font to `fontSize: cq(size)`, the date font to `fontSize: cq(size * 0.26)`, and the date `marginTop` to `cq(size * 0.14)`. Leave the clock logic and `suppressHydrationWarning` unchanged.

- [ ] **Step 2: Update the `@/lib/kiosk-layout` import in `kiosk-preview.tsx`**

Replace the existing import block from `@/lib/kiosk-layout` with:

```tsx
import {
  DEFAULT_KIOSK_LAYOUT,
  type KioskLayout,
  type KioskObject,
} from "@/lib/kiosk-layout";
```

- [ ] **Step 3: Replace `IdleScreen` + `KioskElementView` + `WifiSignal`/`ElementVisual` with object renderers**

Find the current `IdleScreen`, the exported `KioskElementView`, the `ElementVisual` helper, and `WifiSignal`, and replace ALL of them with the following (one definition each; no leftovers). `KioskElementView` is removed and replaced by the exported `ObjectVisual`:

```tsx
/* ── 1 · IDLE / READY (object boxes) ─────────────────────────────────── */
function IdleScreen({ brand, layout }: { brand: KioskBrand; layout: KioskLayout }) {
  const ordered = [...layout.objects].sort((a, b) => a.z - b.z);
  return (
    <div className="absolute inset-0">
      {ordered
        .filter((o) => o.visible)
        .map((o) => (
          <div
            key={o.id}
            style={{
              position: "absolute",
              left: `${o.x * 100}%`,
              top: `${o.y * 100}%`,
              width: `${o.w * 100}%`,
              height: `${o.h * 100}%`,
              zIndex: o.z,
            }}
          >
            <ObjectVisual object={o} brand={brand} layout={layout} />
          </div>
        ))}
    </div>
  );
}

/**
 * Renders one idle object filling its box. Text wraps inside the box at its own
 * font size; logo/clock/wifi size deterministically from the box (no transform
 * scale, no DOM measurement). Shared by the read-only preview and the editor.
 */
export function ObjectVisual({
  object,
  brand,
  layout,
}: {
  object: KioskObject;
  brand: KioskBrand;
  layout: KioskLayout;
}) {
  switch (object.type) {
    case "text":
      return <TextObject object={object} />;
    case "logo":
      return <LogoObject object={object} brand={brand} />;
    case "clock":
      return <ClockObject object={object} layout={layout} />;
    case "wifi":
      return <WifiObject object={object} level={layout.wifiLevel} />;
    default:
      return null;
  }
}

function TextObject({ object }: { object: KioskObject }) {
  const align = object.align ?? "center";
  const justify = align === "left" ? "flex-start" : align === "right" ? "flex-end" : "center";
  return (
    <div
      style={{
        width: "100%",
        height: "100%",
        display: "flex",
        alignItems: "center",
        justifyContent: justify,
        textAlign: align,
        fontSize: cq(object.fontSize ?? 22),
        fontWeight: 600,
        color: "var(--k-fg)",
        lineHeight: 1.15,
        overflow: "hidden",
        overflowWrap: "anywhere",
        whiteSpace: "pre-wrap",
      }}
    >
      {object.text}
    </div>
  );
}

function LogoObject({ object, brand }: { object: KioskObject; brand: KioskBrand }) {
  if (brand.logoUrl) {
    return (
      // eslint-disable-next-line @next/next/no-img-element
      <img src={brand.logoUrl} alt={brand.logoText} style={{ width: "100%", height: "100%", objectFit: "contain" }} />
    );
  }
  // Height-driven so the stacked mark + wordmark fits the box; overflow clipped
  // so an undersized box never lets the logo overlap neighbouring objects.
  const size = object.h * 720 * 0.55;
  return (
    <div style={{ width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center", overflow: "hidden" }}>
      <Logo brand={brand} size={size} stacked />
    </div>
  );
}

function ClockObject({ object, layout }: { object: KioskObject; layout: KioskLayout }) {
  const timeFont = object.h * 720 * 0.5; // time font ~ half the box height
  return (
    <div style={{ width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center", overflow: "hidden" }}>
      <KioskClock timezone={layout.clockTimezone} hour24={layout.clock24h} size={timeFont} />
    </div>
  );
}

function WifiObject({ object, level }: { object: KioskObject; level: number }) {
  const base = Math.min(object.w, object.h) * 720; // fit size in design px
  const bars = [0.45, 0.65, 0.85, 1];
  return (
    <div style={{ width: "100%", height: "100%", display: "flex", alignItems: "flex-end", justifyContent: "center", gap: cq(base * 0.14), color: "var(--k-muted)" }}>
      {bars.map((bh, i) => (
        <span
          key={i}
          style={{
            width: cq(base * 0.22),
            height: cq(base * bh),
            borderRadius: cq(base * 0.1),
            background: "var(--k-fg)",
            opacity: i < level ? 0.85 : 0.2,
          }}
        />
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Verify renderer files type-check + tests**

Run: `npx vitest run` (expect all pass — geometry + layout suites).
Run: `npx tsc --noEmit -p tsconfig.json 2>&1 | grep -E "kiosk-preview|kiosk-clock" || echo "renderer files clean"`
Expected: `renderer files clean` (the editor files still reference the old model — fixed in Tasks 4–5 — so the overall project does not yet type-check; that's expected).

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/kiosk-preview.tsx components/device-preview/kiosk-clock.tsx
git commit -m "feat(kiosk): render idle objects as real boxes (text wrap, box-sized graphics)"
```

---

## Task 4: Editor hook — box drag/resize + snapping (`use-kiosk-editor.ts`)

**Files:**
- Rewrite: `components/device-preview/kiosk-editor/use-kiosk-editor.ts`

- [ ] **Step 1: Replace the file**

```ts
"use client";

import * as React from "react";
import {
  createTextObject,
  defaultLayout,
  MAX_CUSTOM,
  type KioskObject,
  type KioskLayout,
} from "@/lib/kiosk-layout";
import { resizeBox, snapMove, snapResize, clampToCanvas, type Box, type Handle, type Guides } from "@/lib/kiosk-geometry";

const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));
const SNAP = 0.012; // snap threshold (fraction)
const EMPTY_GUIDES: Guides = { vx: [], hy: [] };

const toBox = (o: KioskObject): Box => ({ x: o.x, y: o.y, w: o.w, h: o.h });

type DragKind =
  | { type: "move"; offset: { x: number; y: number } }
  | { type: "resize"; handle: Handle; startBox: Box };

/** Everything the kiosk canvas (KioskStage) and controls (KioskControls) share. */
export interface KioskEditor {
  layout: KioskLayout;
  onChange: (l: KioskLayout) => void;
  disabled: boolean;
  canvasRef: React.RefObject<HTMLDivElement | null>;
  selectedId: string | null;
  setSelectedId: React.Dispatch<React.SetStateAction<string | null>>;
  selected: KioskObject | null;
  selBox: Box | null;
  guides: Guides;
  ordered: KioskObject[];
  atCustomCap: boolean;
  patch: (id: string, p: Partial<KioskObject>) => void;
  startMove: (id: string, e: React.PointerEvent) => void;
  startResize: (handle: Handle, e: React.PointerEvent) => void;
  onPointerMove: (e: React.PointerEvent) => void;
  onPointerUp: (e: React.PointerEvent) => void;
  onCanvasPointerDown: () => void;
  addText: () => void;
  removeObject: (id: string) => void;
  bringToFront: (id: string) => void;
  resetLayout: () => void;
  endInteraction: () => void;
}

export function useKioskEditor({
  layout,
  onChange,
  disabled = false,
}: {
  layout: KioskLayout;
  onChange: (l: KioskLayout) => void;
  disabled?: boolean;
}): KioskEditor {
  const canvasRef = React.useRef<HTMLDivElement>(null);
  const drag = React.useRef<DragKind | null>(null);
  const [selectedId, setSelectedId] = React.useState<string | null>(null);
  const [guides, setGuides] = React.useState<Guides>(EMPTY_GUIDES);

  const selected = layout.objects.find((o) => o.id === selectedId) ?? null;
  const selBox = selected && selected.visible ? toBox(selected) : null;
  const customCount = layout.objects.filter((o) => o.type === "text").length;
  const atCustomCap = customCount >= MAX_CUSTOM;

  function patch(id: string, p: Partial<KioskObject>) {
    onChange({ ...layout, objects: layout.objects.map((o) => (o.id === id ? { ...o, ...p } : o)) });
  }

  function pointerFrac(e: React.PointerEvent): { x: number; y: number } {
    const r = canvasRef.current!.getBoundingClientRect();
    const w = r.width || 1; // guard against a zero-sized canvas producing NaN
    const h = r.height || 1;
    return { x: clamp((e.clientX - r.left) / w, 0, 1), y: clamp((e.clientY - r.top) / h, 0, 1) };
  }

  function others(excludeId: string): Box[] {
    return layout.objects.filter((o) => o.id !== excludeId && o.visible).map(toBox);
  }

  function startMove(id: string, e: React.PointerEvent) {
    if (disabled) return;
    e.stopPropagation();
    e.preventDefault();
    setSelectedId(id);
    const obj = layout.objects.find((o) => o.id === id);
    const p = pointerFrac(e);
    drag.current = { type: "move", offset: obj ? { x: p.x - obj.x, y: p.y - obj.y } : { x: 0, y: 0 } };
    canvasRef.current?.setPointerCapture(e.pointerId);
  }

  function startResize(handle: Handle, e: React.PointerEvent) {
    if (disabled || !selected) return;
    e.stopPropagation();
    e.preventDefault();
    drag.current = { type: "resize", handle, startBox: toBox(selected) };
    canvasRef.current?.setPointerCapture(e.pointerId);
  }

  function onPointerMove(e: React.PointerEvent) {
    const d = drag.current;
    if (!d || !selected) return;
    const p = pointerFrac(e);
    if (d.type === "move") {
      const moved: Box = { x: p.x - d.offset.x, y: p.y - d.offset.y, w: selected.w, h: selected.h };
      const snapped = snapMove(moved, others(selected.id), SNAP);
      const box = clampToCanvas(snapped.box);
      setGuides(snapped.guides);
      patch(selected.id, { x: box.x, y: box.y });
    } else {
      const raw = resizeBox(d.startBox, d.handle, p);
      const snapped = snapResize(raw, d.handle, others(selected.id), SNAP);
      const box = clampToCanvas(snapped.box);
      setGuides(snapped.guides);
      patch(selected.id, { x: box.x, y: box.y, w: box.w, h: box.h });
    }
  }

  function onPointerUp(e: React.PointerEvent) {
    if (drag.current && canvasRef.current?.hasPointerCapture(e.pointerId)) {
      canvasRef.current.releasePointerCapture(e.pointerId);
    }
    drag.current = null;
    setGuides(EMPTY_GUIDES);
  }

  function onCanvasPointerDown() {
    setSelectedId(null);
  }

  function addText() {
    if (disabled || atCustomCap) return;
    const maxZ = layout.objects.reduce((m, o) => Math.max(m, o.z), 0);
    const o = createTextObject("New text", maxZ + 1);
    onChange({ ...layout, objects: [...layout.objects, o] });
    setSelectedId(o.id);
  }

  function removeObject(id: string) {
    onChange({ ...layout, objects: layout.objects.filter((o) => o.id !== id) });
    if (selectedId === id) setSelectedId(null);
  }

  function bringToFront(id: string) {
    const maxZ = layout.objects.reduce((m, o) => Math.max(m, o.z), 0);
    patch(id, { z: maxZ + 1 });
  }

  function resetLayout() {
    onChange(defaultLayout());
    setSelectedId(null);
  }

  /** Clear transient interaction state when the canvas unmounts. */
  function endInteraction() {
    drag.current = null;
    setGuides(EMPTY_GUIDES);
    setSelectedId(null);
  }

  const ordered = [...layout.objects].sort((a, b) => a.z - b.z);

  return {
    layout,
    onChange,
    disabled,
    canvasRef,
    selectedId,
    setSelectedId,
    selected,
    selBox,
    guides,
    ordered,
    atCustomCap,
    patch,
    startMove,
    startResize,
    onPointerMove,
    onPointerUp,
    onCanvasPointerDown,
    addText,
    removeObject,
    bringToFront,
    resetLayout,
    endInteraction,
  };
}
```

- [ ] **Step 2: Verify (hook type-checks against new libs)**

Run: `npx vitest run` (expect all pass).
Run: `npx tsc --noEmit -p tsconfig.json 2>&1 | grep -E "use-kiosk-editor" || echo "hook clean"`
Expected: `hook clean` (the stage/controls still use the old hook shape — fixed in Task 5).

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/kiosk-editor/use-kiosk-editor.ts
git commit -m "feat(kiosk): box-model editor hook (grab-offset move, box resize, snapping)"
```

---

## Task 5: Stage + controls + shell (`kiosk-stage.tsx`, `kiosk-controls.tsx`, `branding-editor.tsx`)

**Files:**
- Rewrite: `components/device-preview/kiosk-editor/kiosk-stage.tsx`
- Rewrite: `components/device-preview/kiosk-editor/kiosk-controls.tsx`
- Modify: `components/branding-editor.tsx`

- [ ] **Step 1: Replace `kiosk-stage.tsx`**

```tsx
"use client";

import * as React from "react";
import { ObjectVisual, kioskRootStyle, type KioskBrand } from "../kiosk-preview";
import { HANDLES, type Box, type Handle } from "@/lib/kiosk-geometry";
import type { KioskEditor } from "./use-kiosk-editor";
import { cn } from "@/lib/utils";

/** The editable kiosk canvas: object boxes, alignment guides, and the
 *  selection/resize overlay. Driven entirely by a useKioskEditor instance. */
export function KioskStage({ editor, brand }: { editor: KioskEditor; brand: KioskBrand }) {
  const { layout, disabled, canvasRef, ordered, guides, selBox, selectedId } = editor;

  // Clear drag/selection when the canvas unmounts (e.g. switching preview screens).
  const endInteraction = editor.endInteraction;
  React.useEffect(() => {
    return () => endInteraction();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div
      ref={canvasRef}
      onPointerMove={editor.onPointerMove}
      onPointerUp={editor.onPointerUp}
      onPointerLeave={editor.onPointerUp}
      onPointerDown={editor.onCanvasPointerDown}
      className="@container relative aspect-square w-full touch-none overflow-hidden rounded-[4cqw] shadow-2xl ring-1 ring-black/10 select-none"
      style={{ ...kioskRootStyle(brand), background: "var(--k-bg)", color: "var(--k-fg)" }}
    >
      {ordered
        .filter((o) => o.visible)
        .map((o) => (
          <div
            key={o.id}
            onPointerDown={(ev) => editor.startMove(o.id, ev)}
            className={cn("absolute", !disabled && "cursor-grab active:cursor-grabbing")}
            style={{ left: `${o.x * 100}%`, top: `${o.y * 100}%`, width: `${o.w * 100}%`, height: `${o.h * 100}%`, zIndex: o.z }}
          >
            <ObjectVisual object={o} brand={brand} layout={layout} />
          </div>
        ))}

      {/* alignment guides */}
      {guides.vx.map((x, i) => (
        <div key={`v${i}`} className="pointer-events-none absolute inset-y-0 w-px" style={{ left: `${x * 100}%`, background: "var(--k-accent)" }} />
      ))}
      {guides.hy.map((y, i) => (
        <div key={`h${i}`} className="pointer-events-none absolute inset-x-0 h-px" style={{ top: `${y * 100}%`, background: "var(--k-accent)" }} />
      ))}

      {/* selection overlay */}
      {selBox && selectedId && !disabled && (
        <SelectionOverlay box={selBox} onResizeStart={editor.startResize} />
      )}
    </div>
  );
}

/** Selection ring + 8 resize handles on the selected object's box. */
function SelectionOverlay({
  box,
  onResizeStart,
}: {
  box: Box;
  onResizeStart: (handle: Handle, e: React.PointerEvent) => void;
}) {
  return (
    <div
      className="pointer-events-none absolute"
      style={{ left: `${box.x * 100}%`, top: `${box.y * 100}%`, width: `${box.w * 100}%`, height: `${box.h * 100}%`, zIndex: 9999 }}
    >
      <div className="absolute -inset-px rounded-[1cqw] ring-2" style={{ "--tw-ring-color": "var(--k-accent)" } as React.CSSProperties} />
      {HANDLES.map((h) => (
        <ResizeHandleDot key={h} handle={h} onDown={(e) => onResizeStart(h, e)} />
      ))}
    </div>
  );
}

const HANDLE_POS: Record<Handle, string> = {
  nw: "left-0 top-0", n: "left-1/2 top-0", ne: "left-full top-0",
  e: "left-full top-1/2", se: "left-full top-full", s: "left-1/2 top-full",
  sw: "left-0 top-full", w: "left-0 top-1/2",
};
const HANDLE_CURSOR: Record<Handle, string> = {
  nw: "nwse-resize", se: "nwse-resize", ne: "nesw-resize", sw: "nesw-resize",
  n: "ns-resize", s: "ns-resize", e: "ew-resize", w: "ew-resize",
};

/** A single resize handle dot, positioned on the box edge by compass name. */
function ResizeHandleDot({ handle, onDown }: { handle: Handle; onDown: (e: React.PointerEvent) => void }) {
  return (
    <div
      onPointerDown={onDown}
      className={cn("pointer-events-auto absolute size-2.5 -translate-x-1/2 -translate-y-1/2 rounded-full border-2 border-white shadow", HANDLE_POS[handle])}
      style={{ cursor: HANDLE_CURSOR[handle], background: "var(--k-accent)" }}
      aria-label={`Resize ${handle}`}
    />
  );
}
```

- [ ] **Step 2: Replace `kiosk-controls.tsx`**

```tsx
"use client";

import * as React from "react";
import { AlignCenter, AlignLeft, AlignRight, Eye, EyeOff, Plus, RotateCcw, Trash2, Wifi } from "lucide-react";
import {
  objectLabel,
  FONT_MIN,
  FONT_MAX,
  MAX_CUSTOM,
  MAX_TEXT_LEN,
  type KioskObject,
  type TextAlign,
} from "@/lib/kiosk-layout";
import { MIN_BOX } from "@/lib/kiosk-geometry";
import type { KioskEditor } from "./use-kiosk-editor";
import { Label } from "@/components/ui/label";
import { Input } from "@/components/ui/input";
import { Switch } from "@/components/ui/switch";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { TIMEZONES } from "@/lib/timezones";
import { cn } from "@/lib/utils";

const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));
const CANVAS_REF_PX = 720;

/** Object list + a type-aware properties panel for the selected object. */
export function KioskControls({ editor }: { editor: KioskEditor }) {
  const { ordered, disabled, selectedId, setSelectedId, selected, atCustomCap } = editor;

  return (
    <div className="space-y-4">
      <div className="space-y-2 rounded-xl border p-3">
        <div className="flex items-center justify-between">
          <span className="text-sm font-medium">Objects</span>
          <button
            type="button"
            disabled={disabled || atCustomCap}
            onClick={editor.addText}
            title={atCustomCap ? `Limit of ${MAX_CUSTOM} text objects reached` : undefined}
            className="flex items-center gap-1 rounded-md border px-2 py-1 text-xs font-medium transition-colors hover:bg-accent disabled:opacity-50"
          >
            <Plus className="size-3.5" /> Add text
          </button>
        </div>
        {ordered.map((o) => {
          const active = selectedId === o.id;
          return (
            <div
              key={o.id}
              onClick={() => o.visible && setSelectedId(o.id)}
              className={cn("flex items-center gap-3 rounded-lg px-2 py-1.5 transition-colors", active && "bg-accent", o.visible && "cursor-pointer")}
            >
              <button
                type="button"
                disabled={disabled}
                onClick={(ev) => { ev.stopPropagation(); editor.patch(o.id, { visible: !o.visible }); }}
                className="text-muted-foreground hover:text-foreground disabled:opacity-50"
                aria-label={o.visible ? `Hide ${objectLabel(o)}` : `Show ${objectLabel(o)}`}
              >
                {o.visible ? <Eye className="size-4" /> : <EyeOff className="size-4" />}
              </button>
              <span className={cn("flex-1 text-sm font-medium", !o.visible && "text-muted-foreground line-through")}>
                {objectLabel(o)}
              </span>
              {o.type === "text" && (
                <button
                  type="button"
                  disabled={disabled}
                  onClick={(ev) => { ev.stopPropagation(); editor.removeObject(o.id); }}
                  className="text-muted-foreground hover:text-destructive disabled:opacity-50"
                  aria-label={`Delete ${objectLabel(o)}`}
                >
                  <Trash2 className="size-4" />
                </button>
              )}
            </div>
          );
        })}
      </div>

      {selected && <Properties key={selected.id} object={selected} editor={editor} />}

      <button
        type="button"
        disabled={disabled}
        onClick={editor.resetLayout}
        className="flex w-full items-center justify-center gap-2 rounded-lg border py-2 text-sm font-medium text-muted-foreground transition-colors hover:bg-accent hover:text-foreground disabled:opacity-50"
      >
        <RotateCcw className="size-4" /> Reset layout to default
      </button>
    </div>
  );
}

/** Type-aware properties for the selected object. */
function Properties({ object, editor }: { object: KioskObject; editor: KioskEditor }) {
  const { disabled, layout, onChange } = editor;
  const set = (p: Partial<KioskObject>) => editor.patch(object.id, p);

  return (
    <div className="space-y-3 rounded-xl border p-3">
      <div className="flex items-center justify-between">
        <span className="text-sm font-medium">{objectLabel(object)}</span>
        <button type="button" disabled={disabled} onClick={() => editor.bringToFront(object.id)} className="text-xs text-muted-foreground hover:text-foreground disabled:opacity-50">
          Bring to front
        </button>
      </div>

      {object.type === "text" && (
        <>
          <div className="space-y-1.5">
            <Label className="text-xs text-muted-foreground">Text</Label>
            <Input value={object.text ?? ""} disabled={disabled} maxLength={MAX_TEXT_LEN} onChange={(e) => set({ text: e.target.value })} className="h-8" />
          </div>
          <div className="grid grid-cols-2 gap-2">
            <NumberField label="Font size" value={Math.round(object.fontSize ?? 24)} disabled={disabled} onChange={(v) => set({ fontSize: clamp(v, FONT_MIN, FONT_MAX) })} />
            <div className="space-y-1">
              <Label className="text-[10px] text-muted-foreground">Align</Label>
              <div className="flex gap-1">
                {([["left", AlignLeft], ["center", AlignCenter], ["right", AlignRight]] as [TextAlign, typeof AlignLeft][]).map(([a, Icon]) => (
                  <button
                    key={a}
                    type="button"
                    disabled={disabled}
                    onClick={() => set({ align: a })}
                    aria-label={`Align ${a}`}
                    className={cn("flex h-8 flex-1 items-center justify-center rounded-md border transition-colors disabled:opacity-50", (object.align ?? "center") === a ? "border-foreground bg-foreground text-background" : "hover:bg-accent")}
                  >
                    <Icon className="size-4" />
                  </button>
                ))}
              </div>
            </div>
          </div>
        </>
      )}

      <div className="grid grid-cols-4 gap-2">
        <NumberField label="X" value={Math.round(object.x * CANVAS_REF_PX)} disabled={disabled} onChange={(v) => set({ x: clamp(v / CANVAS_REF_PX, 0, 1 - object.w) })} />
        <NumberField label="Y" value={Math.round(object.y * CANVAS_REF_PX)} disabled={disabled} onChange={(v) => set({ y: clamp(v / CANVAS_REF_PX, 0, 1 - object.h) })} />
        <NumberField label="W" value={Math.round(object.w * CANVAS_REF_PX)} disabled={disabled} onChange={(v) => set({ w: clamp(v / CANVAS_REF_PX, MIN_BOX, 1 - object.x) })} />
        <NumberField label="H" value={Math.round(object.h * CANVAS_REF_PX)} disabled={disabled} onChange={(v) => set({ h: clamp(v / CANVAS_REF_PX, MIN_BOX, 1 - object.y) })} />
      </div>
      <p className="text-[10px] text-muted-foreground">Pixels on the 720 × 720 kiosk canvas.</p>

      {object.type === "clock" && (
        <div className="grid gap-3 sm:grid-cols-2">
          <div className="space-y-1.5">
            <Label className="text-xs text-muted-foreground">Timezone</Label>
            <Select value={layout.clockTimezone} onValueChange={(v) => onChange({ ...layout, clockTimezone: v })} disabled={disabled}>
              <SelectTrigger className="h-8 w-full"><SelectValue /></SelectTrigger>
              <SelectContent>
                {TIMEZONES.map((tz) => (<SelectItem key={tz.value} value={tz.value}>{tz.label}</SelectItem>))}
              </SelectContent>
            </Select>
          </div>
          <div className="flex items-center justify-between">
            <Label htmlFor="clock-24h" className="text-xs text-muted-foreground">24-hour</Label>
            <Switch id="clock-24h" checked={layout.clock24h} onCheckedChange={(v) => onChange({ ...layout, clock24h: v })} disabled={disabled} />
          </div>
        </div>
      )}

      {object.type === "wifi" && (
        <div className="space-y-1.5">
          <Label className="flex items-center gap-1.5 text-xs text-muted-foreground"><Wifi className="size-3.5" /> Signal level</Label>
          <div className="flex gap-1.5">
            {[0, 1, 2, 3, 4].map((lvl) => (
              <button
                key={lvl}
                type="button"
                disabled={disabled}
                onClick={() => onChange({ ...layout, wifiLevel: lvl })}
                className={cn("h-8 flex-1 rounded-md border text-xs font-medium transition-colors disabled:opacity-50", layout.wifiLevel === lvl ? "border-foreground bg-foreground text-background" : "hover:bg-accent")}
              >
                {lvl}
              </button>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}

/** Small labelled numeric input. Buffers keystrokes; commits on blur/Enter. */
function NumberField({
  label,
  value,
  disabled,
  onChange,
}: {
  label: string;
  value: number;
  disabled?: boolean;
  onChange: (v: number) => void;
}) {
  const [draft, setDraft] = React.useState<string>(value.toString());
  const [focused, setFocused] = React.useState(false);

  React.useEffect(() => {
    if (!focused) setDraft(value.toString());
  }, [value, focused]);

  const commit = () => {
    const n = Number(draft);
    if (draft.trim() !== "" && Number.isFinite(n)) onChange(n);
    else setDraft(value.toString());
  };

  return (
    <div className="space-y-1">
      <Label className="text-[10px] text-muted-foreground">{label}</Label>
      <Input
        type="number"
        inputMode="numeric"
        value={draft}
        disabled={disabled}
        onFocus={() => setFocused(true)}
        onChange={(e) => setDraft(e.target.value)}
        onBlur={() => { setFocused(false); commit(); }}
        onKeyDown={(e) => { if (e.key === "Enter") { commit(); (e.currentTarget as HTMLInputElement).blur(); } }}
        className="h-8 px-2 font-mono text-xs tabular-nums"
      />
    </div>
  );
}
```

- [ ] **Step 3: Update `branding-editor.tsx` — drop `remeasureKey`**

In `components/branding-editor.tsx`, the `useKioskEditor` call currently passes `remeasureKey: logoPreview`. The new hook has no such argument. Change the call to:

```tsx
  const editor = useKioskEditor({
    layout,
    onChange: setLayout,
    disabled: !canEdit,
  });
```
Leave everything else in `branding-editor.tsx` unchanged (it never referenced `layout.elements`/`layout.objects` directly; `dirty` diffs the whole layout via `JSON.stringify`).

- [ ] **Step 4: Type-check the whole project + tests**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no errors (everything now uses the v2 model). Fix any in-file errors you introduced; if an error points elsewhere, report it.
Run: `npx vitest run`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/kiosk-editor/kiosk-stage.tsx components/device-preview/kiosk-editor/kiosk-controls.tsx components/branding-editor.tsx
git commit -m "feat(kiosk): box-model stage + type-aware properties panel; drop remeasureKey"
```

---

## Task 6: Build + verification

**Files:** none (verification only). Fix defects in the owning file from Tasks 1–5 and re-run.

- [ ] **Step 1: Production build**

Run: `npm run build`
Expected: succeeds, no type errors.

- [ ] **Step 2: Confirm no leftover references to the old model**

Run: `grep -rn "KioskElement\b\|\.elements\b\|\bsx\b\|\bsy\b\|remeasureKey\|KioskElementView\|elementLabel\|createTextElement" components/device-preview components/branding-editor.tsx lib/kiosk-layout.ts lib/kiosk-geometry.ts 2>/dev/null || echo "no stale references"`
Expected: `no stale references` (the old `KioskElement`, `elements`, `sx`/`sy`, `KioskElementView`, `elementLabel`, `createTextElement`, `remeasureKey` are all gone).

- [ ] **Step 3: Live verification (post-deploy, production)**

On the deployed site, sign in (`dana@roastwell.co` / `123456`), open `/tenant/branding`, preview on "Idle / ready". Verify, WITHOUT saving except one deliberate round-trip:
- **Drag has no jump** — grabbing an object anywhere moves it smoothly from that grab point (no center-snap teleport).
- **Resize changes a real box** — dragging edges/corners resizes the box; text re-wraps inside without distortion; the selection ring/handles stay crisp; logo/clock/wifi scale to fit their box.
- **Snapping** — moving/resizing an object near the canvas center/edges or another object shows accent guide lines and snaps into alignment.
- **Editable text** — selecting the Tagline or Lane text object shows a Text field; editing updates the canvas; Font size and Align (left/center/right) work.
- **Type-aware panel** — selecting Clock shows timezone + 24-hour; selecting Wi-Fi shows the signal level; X/Y/W/H number boxes reflect and drive the selected object.
- **Add text / delete** — Add text creates an editable object; trash removes text objects; fixed widgets (Logo/Clock/Wi-Fi) show no trash.
- **Save round-trip** — make a change (save bar → "Unsaved changes"), Save once, reload → layout persists. **Note:** on first load after deploy, any old (v1) layout resets to the new default — expected.
- **View-only** — a member-role user sees disabled controls and a disabled (but visible) save bar.

- [ ] **Step 4: Commit any fixes** (skip if none)

```bash
git add -A
git commit -m "fix(kiosk): address issues found in verification"
```

---

## Self-review notes (coverage map)

- Spec §1 object model → Task 2 (`KioskObject`, v2, fixed+text, top-left boxes, helpers, normalize/reset).
- Spec §2 geometry → Task 1 (top-left `resizeBox`, `snapMove`/`snapResize`, `clampToCanvas`, `Guides`).
- Spec §3 rendering → Task 3 (box positioning, `TextObject` wrap, `LogoObject` contain, `ClockObject`/`WifiObject` box-sized, `KioskClock` size prop). No transform scale; no DOM measurement.
- Spec §4 editor → Task 4 (hook: grab-offset move, box resize, snapping, derived `selBox`, guides, no `elRefs`) + Task 5 (stage boxes + overlay + guide lines; type-aware properties panel; shell `remeasureKey` removed).
- Spec §5 persistence/migration/testing → Tasks 1–2 tests; Task 2 normalize v1→default; Task 6 build + grep + live verify.
- Type consistency: `KioskObject`/`KioskLayout`/`defaultLayout`/`createTextObject`/`objectLabel`/`FONT_MIN`/`FONT_MAX`/`MAX_CUSTOM`/`MAX_TEXT_LEN` (Task 2) consumed by rendering (Task 3: `ObjectVisual`), hook (Task 4), controls (Task 5). `Box`/`Handle`/`Guides`/`resizeBox`/`snapMove`/`snapResize`/`clampToCanvas`/`MIN_BOX`/`HANDLES` (Task 1) consumed by hook (Task 4) and stage/controls (Task 5). `KioskEditor` (Task 4) consumed by `KioskStage`/`KioskControls` (Task 5) and instantiated in `branding-editor.tsx`. `ObjectVisual` exported from `kiosk-preview` (Task 3) consumed by `KioskStage` (Task 5). No DB/persistence/schema change.
```
