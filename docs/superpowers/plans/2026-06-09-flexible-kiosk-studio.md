# Flexible Kiosk Studio Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the kiosk idle-screen editor into a freeform tool where users add/remove custom text elements, type exact pixel size & position into number boxes, and resize elements with Word-style drag handles.

**Architecture:** Element size becomes a unitless multiplier of each element's natural rendered size (`sx, sy`; `1,1` = natural). Rendering is a plain `transform: scale(sx, sy)` (no measurement). The editor adds selection, an 8-handle overlay, and an inspector panel; a pure `resizeBox` geometry helper does the handle math in canvas-fraction space; pixel readouts are projected from a live `getBoundingClientRect` measurement against the square canvas. The element id set opens up so custom `text` elements coexist with the 5 built-ins. Storage stays jsonb (no DB migration); `normalizeKioskLayout` guards every load and migrates legacy `scale`.

**Tech Stack:** Next.js 16 / React 19 / TypeScript (strict), Tailwind v4 + shadcn (radix-nova), Vitest, lucide-react. Container-query (`cqw`) sized canvas.

---

## File structure

- **Create** `lib/kiosk-geometry.ts` — pure box/handle geometry: `Box` type, `HANDLES`, `resizeBox()`, `clampCenter()`. No React, no DOM. Fully unit-tested.
- **Create** `lib/kiosk-geometry.test.ts` — unit tests for the geometry helper.
- **Modify** `lib/kiosk-layout.ts` — new `KioskElement` shape (`kind`, `builtin`, `text`, `sx`, `sy`, `z`; drop `scale`), open id set, `createTextElement()`, `elementLabel()`, reworked `normalizeKioskLayout` with legacy migration.
- **Modify** `lib/kiosk-layout.test.ts` — rewrite tests for the new model.
- **Modify** `components/device-preview/kiosk-clock.tsx` — drop the `scale` prop; render at natural size.
- **Modify** `components/device-preview/kiosk-preview.tsx` — `KioskElementView` switches on `kind`/`builtin`, renders natural content wrapped in `transform: scale(sx,sy)`; `IdleScreen` sorts by `z` and positions by center; add a `CustomText` visual.
- **Modify** `components/device-preview/kiosk-layout-editor.tsx` — selection model, handle overlay, drag-move + drag-resize via `resizeBox`, inspector panel with px number inputs, "Add text" button, per-kind controls, delete-custom, z-order, reset.
- **No change needed** to `components/branding-editor.tsx`, `app/(tenant)/tenant/branding/actions.ts`, `app/(tenant)/tenant/branding/page.tsx`, or the DB (verified in Task 6). They pass `layout` opaquely and normalize on save/load.

**Build note:** changing the `KioskElement` shape (Task 2) makes the components stop type-checking until Tasks 3–5 land. Intermediate tasks are verified with **Vitest** (`npx vitest run <file>`), which transpiles without a full project type-check. The first full `npm run build` / `npx tsc --noEmit` happens in Task 5 and must pass there.

---

## Task 1: Pure geometry helper (`lib/kiosk-geometry.ts`)

Resize math in canvas-fraction space, independent of React/DOM so it can be unit-tested. A `Box` is the element's **visual** rectangle as fractions of the square canvas: `cx,cy` = center, `w,h` = width/height (0..1).

**Files:**
- Create: `lib/kiosk-geometry.ts`
- Test: `lib/kiosk-geometry.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// lib/kiosk-geometry.test.ts
import { describe, it, expect } from "vitest";
import { resizeBox, clampCenter, MIN_BOX, type Box } from "./kiosk-geometry";

const box: Box = { cx: 0.5, cy: 0.5, w: 0.2, h: 0.1 }; // edges: L0.4 R0.6 T0.45 B0.55

describe("resizeBox", () => {
  it("right edge moves only width, left edge stays fixed", () => {
    const r = resizeBox(box, "e", { x: 0.7, y: 0.99 }, false);
    expect(r.w).toBeCloseTo(0.3, 6);   // L0.4 → R0.7
    expect(r.h).toBeCloseTo(0.1, 6);   // unchanged
    expect(r.cx).toBeCloseTo(0.55, 6); // midpoint of 0.4..0.7
    expect(r.cy).toBeCloseTo(0.5, 6);
  });

  it("bottom edge moves only height, top stays fixed", () => {
    const r = resizeBox(box, "s", { x: 0.99, y: 0.75 }, false);
    expect(r.h).toBeCloseTo(0.3, 6);   // T0.45 → B0.75
    expect(r.w).toBeCloseTo(0.2, 6);
    expect(r.cy).toBeCloseTo(0.6, 6);
  });

  it("corner keeps aspect ratio (driven by width), opposite corner fixed", () => {
    // se corner: anchor = top-left (0.4, 0.45). aspect w/h = 2.
    const r = resizeBox(box, "se", { x: 0.8, y: 0.99 }, true);
    expect(r.w).toBeCloseTo(0.4, 6);   // 0.4 → 0.8
    expect(r.h).toBeCloseTo(0.2, 6);   // aspect-locked: 0.4 / 2
    expect(r.cx).toBeCloseTo(0.6, 6);  // midpoint 0.4..0.8
    expect(r.cy).toBeCloseTo(0.55, 6); // midpoint 0.45..0.65
  });

  it("clamps to a minimum size and never inverts", () => {
    const r = resizeBox(box, "e", { x: 0.3, y: 0.5 }, false); // dragged past left edge
    expect(r.w).toBeGreaterThanOrEqual(MIN_BOX);
    expect(r.w).toBeLessThanOrEqual(0.2);
  });
});

describe("clampCenter", () => {
  it("keeps the center within [0,1]", () => {
    expect(clampCenter({ cx: -0.2, cy: 1.5, w: 0.1, h: 0.1 })).toMatchObject({ cx: 0, cy: 1 });
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run lib/kiosk-geometry.test.ts`
Expected: FAIL — `Cannot find module './kiosk-geometry'`.

- [ ] **Step 3: Implement the helper**

```ts
// lib/kiosk-geometry.ts
// Pure resize geometry for the kiosk editor. Operates on the element's VISUAL
// rectangle in canvas fractions (0..1); no React/DOM so it is unit-testable.
// The component converts the result back into sx/sy multipliers and x/y center.

export interface Box {
  cx: number; // center x, fraction 0..1
  cy: number; // center y, fraction 0..1
  w: number;  // width, fraction 0..1
  h: number;  // height, fraction 0..1
}

/** The 8 resize handles. n/s/e/w = edges; nw/ne/sw/se = corners. */
export const HANDLES = ["nw", "n", "ne", "e", "se", "s", "sw", "w"] as const;
export type Handle = (typeof HANDLES)[number];

/** Minimum visual size as a fraction of the canvas — keeps elements grabbable. */
export const MIN_BOX = 0.04;

const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));

/**
 * Resize `box` by dragging `handle` to `pointer` (canvas fractions). The edge or
 * corner opposite the handle stays fixed. Corners with `keepAspect` lock the
 * original w:h ratio (driven by the horizontal delta). Result width/height are
 * floored at MIN_BOX and never invert past the fixed anchor.
 */
export function resizeBox(box: Box, handle: Handle, pointer: { x: number; y: number }, keepAspect: boolean): Box {
  let left = box.cx - box.w / 2;
  let right = box.cx + box.w / 2;
  let top = box.cy - box.h / 2;
  let bottom = box.cy + box.h / 2;

  const movesE = handle === "e" || handle === "ne" || handle === "se";
  const movesW = handle === "w" || handle === "nw" || handle === "sw";
  const movesS = handle === "s" || handle === "se" || handle === "sw";
  const movesN = handle === "n" || handle === "ne" || handle === "nw";

  if (movesE) right = Math.max(left + MIN_BOX, pointer.x);
  if (movesW) left = Math.min(right - MIN_BOX, pointer.x);
  if (movesS) bottom = Math.max(top + MIN_BOX, pointer.y);
  if (movesN) top = Math.min(bottom - MIN_BOX, pointer.y);

  let w = right - left;
  let h = bottom - top;

  // Corner drag: lock aspect ratio, driven by the new width.
  const isCorner = handle.length === 2;
  if (isCorner && keepAspect && box.h > 0) {
    const aspect = box.w / box.h;
    h = Math.max(MIN_BOX, w / aspect);
    // Re-anchor height to the fixed vertical edge.
    if (movesS) bottom = top + h;
    if (movesN) top = bottom - h;
  }

  return { cx: (left + right) / 2, cy: (top + bottom) / 2, w, h };
}

/** Keep an element's center on-canvas (size may overhang the edges). */
export function clampCenter(box: Box): Box {
  return { ...box, cx: clamp(box.cx, 0, 1), cy: clamp(box.cy, 0, 1) };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run lib/kiosk-geometry.test.ts`
Expected: PASS (4 + 1 assertions across 5 tests).

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-geometry.ts lib/kiosk-geometry.test.ts
git commit -m "feat(kiosk): pure resizeBox geometry helper for editor handles"
```

---

## Task 2: Data model rework (`lib/kiosk-layout.ts`)

Open the id set, replace `scale` with `sx,sy`, add `kind`/`builtin`/`text`/`z`, migrate legacy data, and add `createTextElement` / `elementLabel` helpers.

**Files:**
- Modify: `lib/kiosk-layout.ts`
- Test: `lib/kiosk-layout.test.ts`

- [ ] **Step 1: Rewrite the test file**

```ts
// lib/kiosk-layout.test.ts
import { describe, it, expect } from "vitest";
import {
  normalizeKioskLayout,
  createTextElement,
  elementLabel,
  DEFAULT_KIOSK_LAYOUT,
  BUILTIN_IDS,
  SCALE_MIN,
  SCALE_MAX,
} from "./kiosk-layout";

describe("normalizeKioskLayout", () => {
  it("returns the default layout for null/garbage", () => {
    expect(normalizeKioskLayout(null)).toEqual(DEFAULT_KIOSK_LAYOUT);
    expect(normalizeKioskLayout("nope")).toEqual(DEFAULT_KIOSK_LAYOUT);
    expect(normalizeKioskLayout({})).toEqual(DEFAULT_KIOSK_LAYOUT);
  });

  it("always includes the 5 built-ins exactly once", () => {
    const l = normalizeKioskLayout({ elements: [{ id: "logo", x: 0.1 }, { id: "bogus" }] });
    const builtins = l.elements.filter((e) => e.kind === "builtin").map((e) => e.builtin).sort();
    expect(builtins).toEqual([...BUILTIN_IDS].sort());
    expect(l.elements.filter((e) => e.kind === "builtin")).toHaveLength(5);
  });

  it("migrates legacy `scale` to sx = sy = scale", () => {
    const l = normalizeKioskLayout({ elements: [{ id: "logo", scale: 1.5 }] });
    const logo = l.elements.find((e) => e.builtin === "logo")!;
    expect(logo.sx).toBe(1.5);
    expect(logo.sy).toBe(1.5);
  });

  it("clamps x,y to [0,1] and sx,sy to [SCALE_MIN, SCALE_MAX]", () => {
    const l = normalizeKioskLayout({ elements: [{ id: "logo", x: -5, y: 9, sx: 99, sy: 0.001 }] });
    const logo = l.elements.find((e) => e.builtin === "logo")!;
    expect(logo.x).toBe(0);
    expect(logo.y).toBe(1);
    expect(logo.sx).toBe(SCALE_MAX);
    expect(logo.sy).toBe(SCALE_MIN);
  });

  it("keeps a valid custom text element and round-trips it", () => {
    const raw = { elements: [{ id: "text-abc", kind: "text", text: "Hello", x: 0.3, y: 0.7, sx: 2, sy: 1 }] };
    const l = normalizeKioskLayout(raw);
    const custom = l.elements.find((e) => e.id === "text-abc")!;
    expect(custom).toMatchObject({ id: "text-abc", kind: "text", text: "Hello", x: 0.3, y: 0.7, sx: 2, sy: 1 });
    // round-trip is stable
    expect(normalizeKioskLayout(l)).toEqual(l);
  });

  it("drops a custom element with no/invalid text and trims long text", () => {
    const l = normalizeKioskLayout({
      elements: [
        { id: "text-bad", kind: "text" },                         // no text → dropped
        { id: "text-long", kind: "text", text: "x".repeat(200) }, // trimmed
      ],
    });
    expect(l.elements.find((e) => e.id === "text-bad")).toBeUndefined();
    expect(l.elements.find((e) => e.id === "text-long")!.text!.length).toBe(80);
  });

  it("caps custom text elements at 20", () => {
    const many = Array.from({ length: 30 }, (_, i) => ({ id: `text-${i}`, kind: "text", text: `t${i}` }));
    const l = normalizeKioskLayout({ elements: many });
    expect(l.elements.filter((e) => e.kind === "text")).toHaveLength(20);
  });

  it("falls back to UTC for an unknown timezone and clamps wifi 0..4", () => {
    expect(normalizeKioskLayout({ clockTimezone: "Mars/Phobos" }).clockTimezone).toBe("UTC");
    expect(normalizeKioskLayout({ clockTimezone: "America/New_York" }).clockTimezone).toBe("America/New_York");
    expect(normalizeKioskLayout({ wifiLevel: 9 }).wifiLevel).toBe(4);
    expect(normalizeKioskLayout({ wifiLevel: -3 }).wifiLevel).toBe(0);
  });
});

describe("createTextElement", () => {
  it("makes a centered text element with a unique text-* id", () => {
    const a = createTextElement("Hi", 5);
    expect(a.kind).toBe("text");
    expect(a.id).toMatch(/^text-/);
    expect(a).toMatchObject({ text: "Hi", x: 0.5, y: 0.5, sx: 1, sy: 1, visible: true, z: 5 });
    expect(createTextElement("Hi", 5).id).not.toBe(a.id);
  });
});

describe("elementLabel", () => {
  it("uses the built-in label or the (truncated) custom text", () => {
    const layout = normalizeKioskLayout({});
    expect(elementLabel(layout.elements.find((e) => e.builtin === "clock")!)).toBe("Clock");
    expect(elementLabel(createTextElement("A very long custom label here", 0))).toContain("A very long");
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run lib/kiosk-layout.test.ts`
Expected: FAIL — exports `createTextElement`, `elementLabel`, `BUILTIN_IDS`, `SCALE_MIN/MAX` and the new shape don't exist yet.

- [ ] **Step 3: Rewrite `lib/kiosk-layout.ts`**

```ts
// Kiosk idle-screen layout: which elements show, where (fractional 0..1 center
// anchors so positions are resolution-independent on the scaling 720² canvas),
// and how big (sx/sy multipliers of each element's NATURAL size; 1 = natural,
// sx≠sy = free-stretch). Built-in elements are a fixed set; users may also add
// custom text elements (open id set). Persisted per tenant as jsonb; always
// loaded through normalizeKioskLayout so malformed/old data can never break the
// render (it also migrates the legacy single `scale` field to sx=sy=scale).
import { isValidTimezone } from "./timezones";

export const BUILTIN_IDS = ["logo", "clock", "wifi", "lane", "tagline"] as const;
export type BuiltinId = (typeof BUILTIN_IDS)[number];

export const KIOSK_ELEMENT_LABEL: Record<BuiltinId, string> = {
  logo: "Logo",
  clock: "Clock",
  wifi: "Wi-Fi signal",
  lane: "Lane label",
  tagline: "Tagline",
};

export interface KioskElement {
  id: string;             // builtin id, or "text-<rand>" for custom
  kind: "builtin" | "text";
  builtin?: BuiltinId;    // present when kind === "builtin"
  text?: string;          // present when kind === "text"
  visible: boolean;
  x: number;              // center anchor, fraction 0..1
  y: number;
  sx: number;             // width multiplier of natural size (1 = natural)
  sy: number;             // height multiplier of natural size
  z: number;              // stacking order
}

export interface KioskLayout {
  version: 1;
  clockTimezone: string; // IANA
  clock24h: boolean;
  wifiLevel: number; // 0..4
  elements: KioskElement[];
}

export const SCALE_MIN = 0.2;
export const SCALE_MAX = 6;
export const MAX_CUSTOM = 20;
export const MAX_TEXT_LEN = 80;

const DEFAULT_BUILTIN: Record<BuiltinId, { visible: boolean; x: number; y: number }> = {
  lane: { visible: true, x: 0.27, y: 0.085 },
  wifi: { visible: true, x: 0.9, y: 0.085 },
  logo: { visible: true, x: 0.5, y: 0.4 },
  clock: { visible: true, x: 0.5, y: 0.62 },
  tagline: { visible: true, x: 0.5, y: 0.93 },
};

export const DEFAULT_KIOSK_LAYOUT: KioskLayout = {
  version: 1,
  clockTimezone: "UTC",
  clock24h: false,
  wifiLevel: 3,
  elements: BUILTIN_IDS.map((id, i) => ({
    id,
    kind: "builtin" as const,
    builtin: id,
    visible: DEFAULT_BUILTIN[id].visible,
    x: DEFAULT_BUILTIN[id].x,
    y: DEFAULT_BUILTIN[id].y,
    sx: 1,
    sy: 1,
    z: i,
  })),
};

const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));

function num(v: unknown, fallback: number): number {
  return typeof v === "number" && Number.isFinite(v) ? v : fallback;
}

/** Read sx/sy, falling back to a legacy single `scale`, then to 1. */
function readScale(e: Record<string, unknown>, axis: "sx" | "sy"): number {
  const legacy = e.scale;
  const v = num(e[axis], num(legacy, 1));
  return clamp(v, SCALE_MIN, SCALE_MAX);
}

/** A fresh custom text element, centered at natural size, on top (`z`). */
export function createTextElement(text: string, z: number): KioskElement {
  const rand = typeof crypto !== "undefined" && crypto.randomUUID
    ? crypto.randomUUID().slice(0, 8)
    : Math.floor(Math.random() * 1e9).toString(36);
  return {
    id: `text-${rand}`,
    kind: "text",
    text: text.slice(0, MAX_TEXT_LEN),
    visible: true,
    x: 0.5,
    y: 0.5,
    sx: 1,
    sy: 1,
    z,
  };
}

/** Display name for the element list / inspector. */
export function elementLabel(e: KioskElement): string {
  if (e.kind === "builtin" && e.builtin) return KIOSK_ELEMENT_LABEL[e.builtin];
  const t = (e.text ?? "").trim();
  return t ? (t.length > 18 ? `${t.slice(0, 18)}…` : t) : "Text";
}

/**
 * Coerce arbitrary stored/loaded data into a valid KioskLayout: all 5 built-ins
 * present exactly once (re-added if missing), plus up to MAX_CUSTOM valid custom
 * text elements. Coords clamped to [0,1], sx/sy to [SCALE_MIN,SCALE_MAX], a known
 * timezone, wifi 0..4. Legacy `scale` migrates to sx=sy=scale. Unknown ids and
 * text-less custom elements are dropped.
 */
export function normalizeKioskLayout(raw: unknown): KioskLayout {
  const r = (raw ?? {}) as Partial<KioskLayout> & { elements?: unknown };
  const list = Array.isArray(r.elements) ? (r.elements as Record<string, unknown>[]) : [];
  const byBuiltin = new Map<BuiltinId, Record<string, unknown>>();
  for (const e of list) {
    const id = e?.id as string;
    if (BUILTIN_IDS.includes(id as BuiltinId)) byBuiltin.set(id as BuiltinId, e);
  }

  // 1) Built-ins, in their default z order.
  const elements: KioskElement[] = BUILTIN_IDS.map((id, i) => {
    const d = DEFAULT_BUILTIN[id];
    const e = byBuiltin.get(id) ?? {};
    return {
      id,
      kind: "builtin" as const,
      builtin: id,
      visible: typeof e.visible === "boolean" ? e.visible : d.visible,
      x: clamp(num(e.x, d.x), 0, 1),
      y: clamp(num(e.y, d.y), 0, 1),
      sx: readScale(e, "sx"),
      sy: readScale(e, "sy"),
      z: typeof e.z === "number" && Number.isFinite(e.z) ? e.z : i,
    };
  });

  // 2) Custom text elements (open id set), capped.
  let zNext = BUILTIN_IDS.length;
  let kept = 0;
  for (const e of list) {
    if (kept >= MAX_CUSTOM) break;
    const id = e?.id as string;
    if (!id || BUILTIN_IDS.includes(id as BuiltinId)) continue;
    if (e.kind !== "text" || typeof e.text !== "string" || e.text.trim() === "") continue;
    elements.push({
      id,
      kind: "text",
      text: e.text.slice(0, MAX_TEXT_LEN),
      visible: typeof e.visible === "boolean" ? e.visible : true,
      x: clamp(num(e.x, 0.5), 0, 1),
      y: clamp(num(e.y, 0.5), 0, 1),
      sx: readScale(e, "sx"),
      sy: readScale(e, "sy"),
      z: typeof e.z === "number" && Number.isFinite(e.z) ? e.z : zNext++,
    });
    kept++;
  }

  const tz = typeof r.clockTimezone === "string" && isValidTimezone(r.clockTimezone)
    ? r.clockTimezone
    : DEFAULT_KIOSK_LAYOUT.clockTimezone;

  return {
    version: 1,
    clockTimezone: tz,
    clock24h: typeof r.clock24h === "boolean" ? r.clock24h : false,
    wifiLevel: clamp(Math.round(num(r.wifiLevel, 3)), 0, 4),
    elements,
  };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run lib/kiosk-layout.test.ts`
Expected: PASS (all describe blocks green).

- [ ] **Step 5: Commit**

```bash
git add lib/kiosk-layout.ts lib/kiosk-layout.test.ts
git commit -m "feat(kiosk): sx/sy size model, open id set, custom text + legacy migration"
```

---

## Task 3: Renderer — multiplier-driven elements (`kiosk-preview.tsx` + `kiosk-clock.tsx`)

Render every idle element at natural size wrapped in `transform: scale(sx, sy)`, switching on `kind`/`builtin`. Add a custom-text visual. Drop the now-unused `scale` prop from `KioskClock`.

**Files:**
- Modify: `components/device-preview/kiosk-clock.tsx`
- Modify: `components/device-preview/kiosk-preview.tsx:191-294` (IdleScreen, KioskElementView, WifiSignal)

- [ ] **Step 1: Drop `scale` from `KioskClock`**

In `components/device-preview/kiosk-clock.tsx`, change the signature and remove `scale` from the font sizes (render natural; the wrapper transform handles sizing):

```tsx
export function KioskClock({
  timezone,
  hour24 = false,
}: {
  timezone: string;
  hour24?: boolean;
}) {
```

Then replace the three `cq(... * scale)` usages with the natural sizes:

```tsx
        fontSize: cq(84),
```
```tsx
        <div style={{ fontSize: cq(22), fontWeight: 500, color: "var(--k-muted)", marginTop: cq(12) }}>
```

(Leave the rest of the component unchanged.)

- [ ] **Step 2: Rewrite `IdleScreen` and `KioskElementView` in `kiosk-preview.tsx`**

Replace the current `IdleScreen` (lines ~191-214), `KioskElementView` (~220-272), and `WifiSignal` (~275-294) with:

```tsx
/* ── 1 · IDLE / READY (modular: free-positioned, sx/sy-sized elements) ── */
function IdleScreen({ brand, layout }: { brand: KioskBrand; layout: KioskLayout }) {
  const ordered = [...layout.elements].sort((a, b) => a.z - b.z);
  return (
    <div className="absolute inset-0">
      {ordered
        .filter((el) => el.visible)
        .map((el) => (
          <div
            key={el.id}
            style={{
              position: "absolute",
              left: `${el.x * 100}%`,
              top: `${el.y * 100}%`,
              transform: "translate(-50%, -50%)",
              zIndex: el.z,
            }}
          >
            <KioskElementView element={el} brand={brand} layout={layout} />
          </div>
        ))}
    </div>
  );
}

/**
 * Renders a single idle-screen element. The visual is drawn at its NATURAL size
 * and scaled by the element's sx/sy multipliers (sx=sy=1 → natural; sx≠sy →
 * free-stretch). Shared by the read-only preview and the drag studio so both
 * look identical.
 */
export function KioskElementView({
  element,
  brand,
  layout,
}: {
  element: KioskElement;
  brand: KioskBrand;
  layout: KioskLayout;
}) {
  return (
    <div style={{ transform: `scale(${element.sx}, ${element.sy})`, transformOrigin: "center" }}>
      <ElementVisual element={element} brand={brand} layout={layout} />
    </div>
  );
}

/** The natural-size visual for an element, before sx/sy scaling. */
function ElementVisual({
  element,
  brand,
  layout,
}: {
  element: KioskElement;
  brand: KioskBrand;
  layout: KioskLayout;
}) {
  if (element.kind === "text") {
    return (
      <div
        style={{
          fontSize: cq(22),
          fontWeight: 600,
          color: "var(--k-fg)",
          letterSpacing: "0.2px",
          textAlign: "center",
          whiteSpace: "pre",
        }}
      >
        {element.text}
      </div>
    );
  }
  switch (element.builtin) {
    case "logo":
      return <Logo brand={brand} size={108} stacked />;
    case "clock":
      return <KioskClock timezone={layout.clockTimezone} hour24={layout.clock24h} />;
    case "wifi":
      return <WifiSignal level={layout.wifiLevel} />;
    case "lane":
      return (
        <div
          style={{
            display: "flex",
            alignItems: "center",
            gap: cq(9),
            fontSize: cq(19),
            fontWeight: 600,
            color: "var(--k-muted)",
            whiteSpace: "nowrap",
          }}
        >
          <span>{brand.storeName}</span>
          <span style={{ opacity: 0.4 }}>·</span>
          <span>{brand.lane ?? "Lane 1"}</span>
        </div>
      );
    case "tagline":
      return (
        <div
          style={{
            fontSize: cq(18),
            fontWeight: 500,
            color: "var(--k-muted)",
            letterSpacing: "0.3px",
            textAlign: "center",
            whiteSpace: "nowrap",
          }}
        >
          Tap your card or pay at the reader to begin
        </div>
      );
    default:
      return null;
  }
}

/** Ascending Wi-Fi signal bars (0–4 filled), natural size. */
function WifiSignal({ level }: { level: number }) {
  const bars = [0.45, 0.65, 0.85, 1];
  const unit = 28; // overall height in design px
  return (
    <div style={{ display: "flex", alignItems: "flex-end", gap: cq(4), color: "var(--k-muted)" }}>
      {bars.map((h, i) => (
        <span
          key={i}
          style={{
            width: cq(7),
            height: cq(unit * h),
            borderRadius: cq(3),
            background: "var(--k-fg)",
            opacity: i < level ? 0.85 : 0.2,
          }}
        />
      ))}
    </div>
  );
}
```

- [ ] **Step 3: Fix the `KioskElement` import in `kiosk-preview.tsx`**

The top import currently pulls `KioskElementId`. Update line ~6-10 to:

```tsx
import {
  DEFAULT_KIOSK_LAYOUT,
  type KioskLayout,
  type KioskElement,
} from "@/lib/kiosk-layout";
```

- [ ] **Step 4: Type-check the renderer files**

Run: `npx vitest run lib/kiosk-layout.test.ts && npx tsc --noEmit -p tsconfig.json 2>&1 | grep -E "kiosk-preview|kiosk-clock" || echo "no errors in renderer files"`
Expected: `no errors in renderer files` (the editor still has errors — fixed in Task 4 — which is expected).

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/kiosk-preview.tsx components/device-preview/kiosk-clock.tsx
git commit -m "feat(kiosk): render idle elements via sx/sy transform + custom text"
```

---

## Task 4: Editor — selection, handles, inspector, add/delete (`kiosk-layout-editor.tsx`)

Rebuild the editor: click-to-select with an 8-handle overlay, drag-to-move, drag-to-resize via `resizeBox`, an inspector with pixel X/Y/W/H inputs and per-kind controls, an "Add text" button, delete for custom elements, and z-order. This is a full rewrite of the file.

**Files:**
- Modify (rewrite): `components/device-preview/kiosk-layout-editor.tsx`

Key mechanics:
- **Canvas-fraction conversion:** `pointerFrac(e)` maps a pointer event to `{x,y}` in 0..1 using the canvas `getBoundingClientRect()` (as today).
- **Visual rect of an element:** read the rendered element node's `getBoundingClientRect()` and divide by the canvas rect to get a `Box` in fractions — this already includes the `scale(sx,sy)` transform, so handles sit on the visible edges.
- **Move:** drag the body → set `x,y` (with center snap guides, as today), clamped via `clampCenter`.
- **Resize:** drag a handle → `resizeBox(currentBox, handle, pointerFrac, keepAspect=isCorner)`, then convert back: `sx *= newBox.w / currentBox.w`, `sy *= newBox.h / currentBox.h`, `x = newBox.cx`, `y = newBox.cy`; clamp `sx,sy` to `[SCALE_MIN, SCALE_MAX]`.
- **Pixel readouts:** measure the selected element's natural size from a hidden node ref + canvas width → `naturalPx720 = naturalOffsetPx * 720 / canvasWidthPx`. Show `W = round(naturalPx720.w * sx)`, `H = round(naturalPx720.h * sy)`, `X = round(x*720)`, `Y = round(y*720)`. Typing converts back: `sx = typedW / naturalPx720.w`, etc.

- [ ] **Step 1: Replace the whole file**

```tsx
"use client";

import * as React from "react";
import { Eye, EyeOff, Plus, RotateCcw, Trash2, Wifi } from "lucide-react";
import { KioskElementView, kioskRootStyle, type KioskBrand } from "./kiosk-preview";
import {
  DEFAULT_KIOSK_LAYOUT,
  createTextElement,
  elementLabel,
  SCALE_MIN,
  SCALE_MAX,
  type KioskElement,
  type KioskLayout,
} from "@/lib/kiosk-layout";
import { HANDLES, resizeBox, clampCenter, type Box, type Handle } from "@/lib/kiosk-geometry";
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
const SNAP = 0.025; // snap-to-center threshold (fraction)
const CANVAS_REF_PX = 720; // design reference for pixel readouts

type DragKind =
  | { type: "move" }
  | { type: "resize"; handle: Handle; startBox: Box; startEl: KioskElement };

export function KioskLayoutEditor({
  brand,
  layout,
  onChange,
  disabled,
}: {
  brand: KioskBrand;
  layout: KioskLayout;
  onChange: (l: KioskLayout) => void;
  disabled?: boolean;
}) {
  const canvasRef = React.useRef<HTMLDivElement>(null);
  const elRefs = React.useRef<Map<string, HTMLDivElement>>(new Map());
  const drag = React.useRef<DragKind | null>(null);
  const [selectedId, setSelectedId] = React.useState<string | null>(null);
  const [guide, setGuide] = React.useState<{ x: boolean; y: boolean }>({ x: false, y: false });
  // The selected element's VISUAL rect (in canvas fractions), measured from the
  // DOM so the handle overlay tracks the scaled size — transforms don't change
  // an element's layout box, so handles can't be children of the scaled node.
  const [selBox, setSelBox] = React.useState<Box | null>(null);

  const selected = layout.elements.find((e) => e.id === selectedId) ?? null;

  function patch(id: string, p: Partial<KioskElement>) {
    onChange({ ...layout, elements: layout.elements.map((e) => (e.id === id ? { ...e, ...p } : e)) });
  }

  function pointerFrac(e: React.PointerEvent): { x: number; y: number } {
    const r = canvasRef.current!.getBoundingClientRect();
    return { x: clamp((e.clientX - r.left) / r.width, 0, 1), y: clamp((e.clientY - r.top) / r.height, 0, 1) };
  }

  /** Element visual rect in canvas fractions (includes the scale transform). */
  function elementBox(id: string): Box {
    const canvas = canvasRef.current!.getBoundingClientRect();
    const node = elRefs.current.get(id)!.getBoundingClientRect();
    return {
      cx: (node.left + node.width / 2 - canvas.left) / canvas.width,
      cy: (node.top + node.height / 2 - canvas.top) / canvas.height,
      w: node.width / canvas.width,
      h: node.height / canvas.height,
    };
  }

  function startMove(id: string, e: React.PointerEvent) {
    if (disabled) return;
    e.stopPropagation();
    e.preventDefault();
    setSelectedId(id);
    drag.current = { type: "move" };
    canvasRef.current?.setPointerCapture(e.pointerId);
  }

  function startResize(handle: Handle, e: React.PointerEvent) {
    if (disabled || !selected) return;
    e.stopPropagation();
    e.preventDefault();
    drag.current = { type: "resize", handle, startBox: elementBox(selected.id), startEl: selected };
    canvasRef.current?.setPointerCapture(e.pointerId);
  }

  function onPointerMove(e: React.PointerEvent) {
    const d = drag.current;
    if (!d || !selected) return;
    const p = pointerFrac(e);
    if (d.type === "move") {
      let x = p.x;
      let y = p.y;
      const snapX = Math.abs(x - 0.5) < SNAP;
      const snapY = Math.abs(y - 0.5) < SNAP;
      if (snapX) x = 0.5;
      if (snapY) y = 0.5;
      setGuide({ x: snapX, y: snapY });
      const c = clampCenter({ cx: x, cy: y, w: 0, h: 0 });
      patch(selected.id, { x: c.cx, y: c.cy });
    } else {
      const isCorner = d.handle.length === 2;
      const nb = clampCenter(resizeBox(d.startBox, d.handle, p, isCorner));
      const sx = clamp(d.startEl.sx * (nb.w / d.startBox.w), SCALE_MIN, SCALE_MAX);
      const sy = clamp(d.startEl.sy * (nb.h / d.startBox.h), SCALE_MIN, SCALE_MAX);
      patch(selected.id, { x: nb.cx, y: nb.cy, sx, sy });
    }
  }

  function onPointerUp(e: React.PointerEvent) {
    if (drag.current) canvasRef.current?.releasePointerCapture(e.pointerId);
    drag.current = null;
    setGuide({ x: false, y: false });
  }

  function addText() {
    if (disabled) return;
    const maxZ = layout.elements.reduce((m, e) => Math.max(m, e.z), 0);
    const el = createTextElement("New text", maxZ + 1);
    onChange({ ...layout, elements: [...layout.elements, el] });
    setSelectedId(el.id);
  }

  function removeEl(id: string) {
    onChange({ ...layout, elements: layout.elements.filter((e) => e.id !== id) });
    if (selectedId === id) setSelectedId(null);
  }

  function bringToFront(id: string) {
    const maxZ = layout.elements.reduce((m, e) => Math.max(m, e.z), 0);
    patch(id, { z: maxZ + 1 });
  }

  // Keep the handle overlay synced to the selected element's measured visual box.
  // Re-measures whenever its geometry/text changes (incl. during a drag, since
  // patch() updates those). Boxes are fractions, so canvas resize needs no recompute.
  React.useLayoutEffect(() => {
    if (!selectedId || !selected?.visible || !elRefs.current.get(selectedId) || !canvasRef.current) {
      setSelBox(null);
      return;
    }
    setSelBox(elementBox(selectedId));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selectedId, selected?.visible, selected?.x, selected?.y, selected?.sx, selected?.sy, selected?.text]);

  const ordered = [...layout.elements].sort((a, b) => a.z - b.z);

  return (
    <div className="space-y-4">
      {/* ── Canvas ── */}
      <div
        ref={canvasRef}
        onPointerMove={onPointerMove}
        onPointerUp={onPointerUp}
        onPointerLeave={onPointerUp}
        onPointerDown={() => setSelectedId(null)}
        className="@container relative aspect-square w-full touch-none overflow-hidden rounded-[4cqw] shadow-2xl ring-1 ring-black/10 select-none"
        style={{ ...kioskRootStyle(brand), background: "var(--k-bg)", color: "var(--k-fg)" }}
      >
        {guide.x && <div className="pointer-events-none absolute inset-y-0 left-1/2 w-px -translate-x-1/2" style={{ background: "var(--k-accent)" }} />}
        {guide.y && <div className="pointer-events-none absolute inset-x-0 top-1/2 h-px -translate-y-1/2" style={{ background: "var(--k-accent)" }} />}

        {ordered
          .filter((e) => e.visible)
          .map((e) => (
            <div
              key={e.id}
              ref={(n) => {
                if (n) elRefs.current.set(e.id, n);
                else elRefs.current.delete(e.id);
              }}
              onPointerDown={(ev) => startMove(e.id, ev)}
              className={cn("absolute", !disabled && "cursor-grab active:cursor-grabbing")}
              style={{ left: `${e.x * 100}%`, top: `${e.y * 100}%`, transform: "translate(-50%, -50%)", zIndex: e.z }}
            >
              <KioskElementView element={e} brand={brand} layout={layout} />
            </div>
          ))}

        {/* Selection overlay (separate layer so handles match the SCALED size). */}
        {selBox && selectedId && !disabled && (
          <SelectionOverlay box={selBox} onResizeStart={startResize} />
        )}
      </div>

      <p className="text-center text-xs text-muted-foreground">
        Click to select · drag to move · drag a handle to resize (corners keep proportions, edges stretch)
      </p>

      {/* ── Add / element list ── */}
      <div className="space-y-2 rounded-xl border p-3">
        <div className="flex items-center justify-between">
          <span className="text-sm font-medium">Elements</span>
          <button
            type="button"
            disabled={disabled}
            onClick={addText}
            className="flex items-center gap-1 rounded-md border px-2 py-1 text-xs font-medium transition-colors hover:bg-accent disabled:opacity-50"
          >
            <Plus className="size-3.5" /> Add text
          </button>
        </div>
        {ordered.map((e) => {
          const active = selectedId === e.id;
          return (
            <div
              key={e.id}
              onClick={() => e.visible && setSelectedId(e.id)}
              className={cn(
                "flex items-center gap-3 rounded-lg px-2 py-1.5 transition-colors",
                active && "bg-accent",
                e.visible && "cursor-pointer",
              )}
            >
              <button
                type="button"
                disabled={disabled}
                onClick={(ev) => { ev.stopPropagation(); patch(e.id, { visible: !e.visible }); }}
                className="text-muted-foreground hover:text-foreground disabled:opacity-50"
                aria-label={e.visible ? `Hide ${elementLabel(e)}` : `Show ${elementLabel(e)}`}
              >
                {e.visible ? <Eye className="size-4" /> : <EyeOff className="size-4" />}
              </button>
              <span className={cn("flex-1 text-sm font-medium", !e.visible && "text-muted-foreground line-through")}>
                {elementLabel(e)}
              </span>
              {e.kind === "text" && (
                <button
                  type="button"
                  disabled={disabled}
                  onClick={(ev) => { ev.stopPropagation(); removeEl(e.id); }}
                  className="text-muted-foreground hover:text-destructive disabled:opacity-50"
                  aria-label={`Delete ${elementLabel(e)}`}
                >
                  <Trash2 className="size-4" />
                </button>
              )}
            </div>
          );
        })}
      </div>

      {/* ── Inspector for the selected element ── */}
      {selected && (
        <Inspector
          key={selected.id}
          el={selected}
          canvasRef={canvasRef}
          elRefs={elRefs}
          disabled={disabled}
          onPatch={(p) => patch(selected.id, p)}
          onBringToFront={() => bringToFront(selected.id)}
        />
      )}

      {/* ── Clock + Wi-Fi ── */}
      <div className="grid gap-3 rounded-xl border p-3 sm:grid-cols-2">
        <div className="space-y-1.5">
          <Label className="text-xs text-muted-foreground">Clock timezone</Label>
          <Select value={layout.clockTimezone} onValueChange={(v) => onChange({ ...layout, clockTimezone: v })} disabled={disabled}>
            <SelectTrigger className="h-8 w-full"><SelectValue /></SelectTrigger>
            <SelectContent>
              {TIMEZONES.map((tz) => (<SelectItem key={tz.value} value={tz.value}>{tz.label}</SelectItem>))}
            </SelectContent>
          </Select>
        </div>
        <div className="flex items-center justify-between">
          <Label htmlFor="clock-24h" className="text-xs text-muted-foreground">24-hour clock</Label>
          <Switch id="clock-24h" checked={layout.clock24h} onCheckedChange={(v) => onChange({ ...layout, clock24h: v })} disabled={disabled} />
        </div>
        <div className="space-y-1.5 sm:col-span-2">
          <Label className="flex items-center gap-1.5 text-xs text-muted-foreground"><Wifi className="size-3.5" /> Wi-Fi signal</Label>
          <div className="flex gap-1.5">
            {[0, 1, 2, 3, 4].map((lvl) => (
              <button
                key={lvl}
                type="button"
                disabled={disabled}
                onClick={() => onChange({ ...layout, wifiLevel: lvl })}
                className={cn(
                  "h-8 flex-1 rounded-md border text-xs font-medium transition-colors disabled:opacity-50",
                  layout.wifiLevel === lvl ? "border-foreground bg-foreground text-background" : "hover:bg-accent",
                )}
              >
                {lvl}
              </button>
            ))}
          </div>
        </div>
      </div>

      <button
        type="button"
        disabled={disabled}
        onClick={() => { onChange(DEFAULT_KIOSK_LAYOUT); setSelectedId(null); }}
        className="flex w-full items-center justify-center gap-2 rounded-lg border py-2 text-sm font-medium text-muted-foreground transition-colors hover:bg-accent hover:text-foreground disabled:opacity-50"
      >
        <RotateCcw className="size-4" /> Reset layout to default
      </button>
    </div>
  );
}

/**
 * Selection ring + 8 resize handles, drawn on its own layer at the element's
 * measured visual box (fractions). The layer is pointer-transparent except the
 * handle dots, so dragging the element body underneath still moves it.
 */
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
      style={{
        left: `${(box.cx - box.w / 2) * 100}%`,
        top: `${(box.cy - box.h / 2) * 100}%`,
        width: `${box.w * 100}%`,
        height: `${box.h * 100}%`,
        zIndex: 9999,
      }}
    >
      <div className="absolute -inset-1 rounded-[1cqw] ring-2" style={{ "--tw-ring-color": "var(--k-accent)" } as React.CSSProperties} />
      {HANDLES.map((h) => (
        <ResizeHandleDot key={h} handle={h} onDown={(e) => onResizeStart(h, e)} />
      ))}
    </div>
  );
}

/** A single resize handle dot, positioned on the overlay edge by compass name. */
function ResizeHandleDot({ handle, onDown }: { handle: Handle; onDown: (e: React.PointerEvent) => void }) {
  const pos: Record<Handle, string> = {
    nw: "left-0 top-0", n: "left-1/2 top-0", ne: "right-0 top-0",
    e: "right-0 top-1/2", se: "right-0 bottom-0", s: "left-1/2 bottom-0",
    sw: "left-0 bottom-0", w: "left-0 top-1/2",
  };
  const cursor: Record<Handle, string> = {
    nw: "nwse-resize", se: "nwse-resize", ne: "nesw-resize", sw: "nesw-resize",
    n: "ns-resize", s: "ns-resize", e: "ew-resize", w: "ew-resize",
  };
  return (
    <div
      onPointerDown={onDown}
      className={cn("pointer-events-auto absolute size-2.5 -translate-x-1/2 -translate-y-1/2 rounded-full border-2 border-white shadow", pos[handle])}
      style={{ cursor: cursor[handle], background: "var(--k-accent)" }}
      aria-label={`Resize ${handle}`}
    />
  );
}

/** Inspector: pixel X/Y/W/H number boxes + per-kind controls for one element. */
function Inspector({
  el,
  canvasRef,
  elRefs,
  disabled,
  onPatch,
  onBringToFront,
}: {
  el: KioskElement;
  canvasRef: React.RefObject<HTMLDivElement | null>;
  elRefs: React.RefObject<Map<string, HTMLDivElement>>;
  disabled?: boolean;
  onPatch: (p: Partial<KioskElement>) => void;
  onBringToFront: () => void;
}) {
  // Natural pixel size on the 720 reference (visual size ÷ current sx/sy).
  const [natural, setNatural] = React.useState<{ w: number; h: number } | null>(null);
  React.useLayoutEffect(() => {
    const canvas = canvasRef.current;
    const node = elRefs.current?.get(el.id);
    if (!canvas || !node) return;
    const cw = canvas.getBoundingClientRect().width || 1;
    const r = node.getBoundingClientRect();
    const k = CANVAS_REF_PX / cw;
    setNatural({ w: (r.width * k) / el.sx, h: (r.height * k) / el.sy });
  }, [el.id, el.sx, el.sy, el.text, canvasRef, elRefs]);

  const wPx = natural ? Math.round(natural.w * el.sx) : null;
  const hPx = natural ? Math.round(natural.h * el.sy) : null;

  const setWidthPx = (v: number) => { if (natural && natural.w > 0) onPatch({ sx: clamp(v / natural.w, SCALE_MIN, SCALE_MAX) }); };
  const setHeightPx = (v: number) => { if (natural && natural.h > 0) onPatch({ sy: clamp(v / natural.h, SCALE_MIN, SCALE_MAX) }); };

  return (
    <div className="space-y-3 rounded-xl border p-3">
      <div className="flex items-center justify-between">
        <span className="text-sm font-medium">{elementLabel(el)}</span>
        <button type="button" disabled={disabled} onClick={onBringToFront} className="text-xs text-muted-foreground hover:text-foreground disabled:opacity-50">
          Bring to front
        </button>
      </div>

      {el.kind === "text" && (
        <div className="space-y-1.5">
          <Label className="text-xs text-muted-foreground">Text</Label>
          <Input value={el.text ?? ""} disabled={disabled} maxLength={80} onChange={(e) => onPatch({ text: e.target.value })} className="h-8" />
        </div>
      )}

      <div className="grid grid-cols-4 gap-2">
        <NumberField label="X" value={Math.round(el.x * CANVAS_REF_PX)} disabled={disabled} onChange={(v) => onPatch({ x: clamp(v / CANVAS_REF_PX, 0, 1) })} />
        <NumberField label="Y" value={Math.round(el.y * CANVAS_REF_PX)} disabled={disabled} onChange={(v) => onPatch({ y: clamp(v / CANVAS_REF_PX, 0, 1) })} />
        <NumberField label="W" value={wPx} disabled={disabled || !natural} onChange={setWidthPx} />
        <NumberField label="H" value={hPx} disabled={disabled || !natural} onChange={setHeightPx} />
      </div>
      <p className="text-[10px] text-muted-foreground">Pixels on the 720 × 720 kiosk canvas.</p>
    </div>
  );
}

/** Small labelled numeric input (px). */
function NumberField({
  label,
  value,
  disabled,
  onChange,
}: {
  label: string;
  value: number | null;
  disabled?: boolean;
  onChange: (v: number) => void;
}) {
  return (
    <div className="space-y-1">
      <Label className="text-[10px] text-muted-foreground">{label}</Label>
      <Input
        type="number"
        inputMode="numeric"
        value={value ?? ""}
        disabled={disabled}
        onChange={(e) => {
          const n = Number(e.target.value);
          if (Number.isFinite(n)) onChange(n);
        }}
        className="h-8 px-2 font-mono text-xs tabular-nums"
      />
    </div>
  );
}
```

- [ ] **Step 2: Type-check the whole project**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: PASS (no errors). If `KioskElementView`'s old `id`/`scale` props are referenced anywhere else, fix those call sites — they should not exist outside these files.

- [ ] **Step 3: Run the full unit suite**

Run: `npx vitest run`
Expected: PASS (all tests, including kiosk-layout + kiosk-geometry).

- [ ] **Step 4: Commit**

```bash
git add components/device-preview/kiosk-layout-editor.tsx
git commit -m "feat(kiosk): editor with resize handles, px inspector, add/delete custom text"
```

---

## Task 5: Build + live verification

Confirm the whole feature compiles, the existing save/load plumbing carries the richer layout unchanged, and the editor behaves in a real browser.

**Files:** none (verification only). If a defect is found, fix it in the owning file from Tasks 2–4 and re-run.

- [ ] **Step 1: Production build**

Run: `npm run build`
Expected: build succeeds with no type errors.

- [ ] **Step 2: Confirm no other call sites broke**

Run: `grep -rn "KioskElementId\|\.scale" components/device-preview app/\(tenant\)/tenant/branding lib/kiosk-layout.ts`
Expected: no references to the removed `KioskElementId` type and no `.scale` on kiosk elements. (`branding-editor.tsx` / `actions.ts` only handle `layout` opaquely — confirm they are untouched and still pass `JSON.stringify(layout)`.)

- [ ] **Step 3: Live browser verification (dev)**

Start dev (`npm run dev`) and sign in as `dana@roastwell.co` / `password123`, go to `/tenant/branding`, select the **Idle / ready** preview screen, then use the browse/Playwright tooling to verify:
- Selecting an element shows the ring + 8 handles.
- Dragging the body moves it; center snap guides appear near the middle.
- Dragging a **corner** scales proportionally; dragging an **edge** stretches one axis.
- The inspector X/Y/W/H boxes reflect the element and editing a box moves/resizes it.
- "Add text" creates a selected, editable text element; editing its text updates the canvas; the trash icon deletes it; built-ins show no trash icon (only the eye toggle).
- "Reset layout to default" restores the 5 built-ins at defaults.

- [ ] **Step 4: Round-trip persistence**

Make a layout change (move + resize an element, add a custom text), click **Save branding**, confirm the success toast, reload the page, and confirm the layout persisted exactly. (Saves via the unchanged `saveBranding` action → `normalizeKioskLayout` → jsonb.)

- [ ] **Step 5: Commit any fixes**

```bash
git add -A
git commit -m "fix(kiosk): address issues found in live verification"
```

(Skip if no fixes were needed.)

---

## Self-review notes (coverage map)

- Spec §1 data model → Task 2. Spec §2 editor UX (select, handles, inspector px, add/delete, z, reset) → Task 4. Spec §3 rendering (transform scale, custom text, shared view) → Task 3. Spec §4 persistence/edge-cases/testing → geometry+normalize tests in Tasks 1–2, edge clamps in `resizeBox`/`clampCenter`/`normalizeKioskLayout`, live + round-trip checks in Task 5.
- No DB migration (jsonb column reused) — confirmed against `actions.ts` upsert and `tenantSettings.kioskLayout`.
- Type consistency: `KioskElementView({ element })` (Task 3) is consumed with `element=` in both `IdleScreen` (Task 3) and the editor (Task 4); `resizeBox`/`clampCenter`/`Box`/`Handle`/`HANDLES`/`MIN_BOX` (Task 1) are imported in Task 4; `createTextElement`/`elementLabel`/`SCALE_MIN`/`SCALE_MAX`/`BUILTIN_IDS` (Task 2) are imported in Tasks 2-test and 4.
```
