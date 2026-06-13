# Branding Phase 1 — UX Shell Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Branding page's studio shell compact and tactile — a collapsible accordion settings panel, a zoomable live preview, swipe/arrow/dot screen navigation, and double-click-to-edit labels on the canvas — with **no data-model changes**.

**Architecture:** Pure UI work on the existing two-pane `BrandingEditor`. Add two missing shadcn primitives (`Accordion`, `Slider`) built on the project's unified `radix-ui` package. Extract a lightweight pointer-based `PreviewCarousel` (no embla). Zoom is applied as the preview canvas's container **width in px** (not a CSS transform), so the kiosk's `aspect-square` box scales proportionally and `useKioskEditor`'s rect-based drag math stays exact. All pure math (zoom clamping, carousel index/swipe) lives in a tested helper module.

**Tech Stack:** Next.js 16 (App Router), React 19, TypeScript strict, Tailwind v4, `radix-ui` (unified package — same import style as `components/ui/tabs.tsx`), lucide-react, vitest (pure-function tests only; the repo has no component-test harness, so UI tasks verify via `npm run build` + a dev-server smoke check).

**Spec:** `docs/superpowers/specs/2026-06-13-branding-page-improvements-design.md` (Phase 1 section).

---

## File Structure

| File | Create/Modify | Responsibility |
|---|---|---|
| `lib/branding-shell.ts` | Create | Pure helpers: zoom constants + `clampZoom`, carousel `stepIndex`/`swipeStep`. |
| `lib/branding-shell.test.ts` | Create | Vitest unit tests for the above. |
| `components/ui/accordion.tsx` | Create | shadcn Accordion over `radix-ui` (radix-nova style). |
| `components/ui/slider.tsx` | Create | shadcn Slider over `radix-ui`. |
| `components/device-preview/kiosk-editor/use-kiosk-editor.ts` | Modify | Expose `isDragging()` for the carousel gesture guard. |
| `components/device-preview/kiosk-preview.tsx` | Modify | Ensure `cq` is exported (used by the inline editor). |
| `components/device-preview/kiosk-editor/kiosk-stage.tsx` | Modify | Double-click a text object → inline `<textarea>` editor. |
| `components/device-preview/preview-carousel.tsx` | Create | Pointer-based swipe carousel with arrows + dots; zoom-aware slide width. |
| `components/branding-editor.tsx` | Modify | Wire zoom slider + carousel into the preview card; replace the 3 stacked `Section` cards with the accordion. |

Each task produces a self-contained, committable change.

---

### Task 1: Pure UI helpers (`lib/branding-shell.ts`)

**Files:**
- Create: `lib/branding-shell.ts`
- Test: `lib/branding-shell.test.ts`

- [ ] **Step 1: Write the failing test**

Create `lib/branding-shell.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import {
  ZOOM_MIN,
  ZOOM_MAX,
  ZOOM_DEFAULT,
  clampZoom,
  stepIndex,
  swipeStep,
} from "./branding-shell";

describe("clampZoom", () => {
  it("clamps below the minimum", () => {
    expect(clampZoom(10)).toBe(ZOOM_MIN);
  });
  it("clamps above the maximum", () => {
    expect(clampZoom(999)).toBe(ZOOM_MAX);
  });
  it("snaps to the nearest 5% step", () => {
    expect(clampZoom(82)).toBe(80);
    expect(clampZoom(83)).toBe(85);
  });
  it("falls back to the default for non-finite input", () => {
    expect(clampZoom(Number.NaN)).toBe(ZOOM_DEFAULT);
  });
});

describe("stepIndex", () => {
  it("advances forward", () => {
    expect(stepIndex(0, 1, 7)).toBe(1);
  });
  it("wraps forward past the end", () => {
    expect(stepIndex(6, 1, 7)).toBe(0);
  });
  it("wraps backward before the start", () => {
    expect(stepIndex(0, -1, 7)).toBe(6);
  });
  it("is safe for an empty list", () => {
    expect(stepIndex(0, 1, 0)).toBe(0);
  });
});

describe("swipeStep", () => {
  it("returns +1 when swiped left past the threshold", () => {
    expect(swipeStep(-80, 200)).toBe(1);
  });
  it("returns -1 when swiped right past the threshold", () => {
    expect(swipeStep(80, 200)).toBe(-1);
  });
  it("returns 0 for a small drag", () => {
    expect(swipeStep(10, 200)).toBe(0);
  });
  it("is safe for a zero-width frame", () => {
    expect(swipeStep(50, 0)).toBe(0);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm test -- branding-shell`
Expected: FAIL — `Cannot find module './branding-shell'`.

- [ ] **Step 3: Write the implementation**

Create `lib/branding-shell.ts`:

```ts
/** Pure helpers for the Branding studio shell (zoom + screen carousel). */

export const ZOOM_MIN = 50;
export const ZOOM_MAX = 125;
export const ZOOM_STEP = 5;
export const ZOOM_DEFAULT = 80;

/** Reference width (px) of the preview canvas at 100% zoom. */
export const PREVIEW_BASE_PX = 600;

/** Clamp a zoom percentage to [MIN, MAX], snapped to the nearest step. */
export function clampZoom(pct: number): number {
  if (!Number.isFinite(pct)) return ZOOM_DEFAULT;
  const stepped = Math.round(pct / ZOOM_STEP) * ZOOM_STEP;
  return Math.min(ZOOM_MAX, Math.max(ZOOM_MIN, stepped));
}

/** The preview canvas width in px for a given zoom percentage. */
export function zoomToPx(pct: number): number {
  return Math.round((PREVIEW_BASE_PX * clampZoom(pct)) / 100);
}

/** Step an index by dir (-1 | +1) with wrap-around. Safe for len <= 0. */
export function stepIndex(current: number, dir: number, len: number): number {
  if (len <= 0) return 0;
  return (current + dir + len) % len;
}

/**
 * How many screens a horizontal swipe should advance.
 * deltaX < 0 (dragged left) → +1 (next); deltaX > 0 → -1 (prev).
 * Returns 0 unless the drag exceeds `threshold` of the frame width.
 */
export function swipeStep(deltaX: number, width: number, threshold = 0.25): number {
  if (width <= 0) return 0;
  const frac = deltaX / width;
  if (frac <= -threshold) return 1;
  if (frac >= threshold) return -1;
  return 0;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npm test -- branding-shell`
Expected: PASS (all cases green).

- [ ] **Step 5: Commit**

```bash
git add lib/branding-shell.ts lib/branding-shell.test.ts
git commit -m "feat(branding): pure helpers for preview zoom + screen carousel"
```

---

### Task 2: Accordion primitive (`components/ui/accordion.tsx`)

**Files:**
- Create: `components/ui/accordion.tsx`

The repo uses the unified `radix-ui` package (see `components/ui/tabs.tsx`). Radix's `Accordion` ships in it, so **no new dependency**. Animation classes are intentionally omitted (the repo has no `tailwindcss-animate` keyframes); the accordion shows/hides without a height animation, which is correct and consistent.

- [ ] **Step 1: Create the component**

Create `components/ui/accordion.tsx`:

```tsx
"use client";

import * as React from "react";
import { Accordion as AccordionPrimitive } from "radix-ui";
import { ChevronDownIcon } from "lucide-react";

import { cn } from "@/lib/utils";

function Accordion(props: React.ComponentProps<typeof AccordionPrimitive.Root>) {
  return <AccordionPrimitive.Root data-slot="accordion" {...props} />;
}

function AccordionItem({
  className,
  ...props
}: React.ComponentProps<typeof AccordionPrimitive.Item>) {
  return (
    <AccordionPrimitive.Item
      data-slot="accordion-item"
      className={cn("border-b last:border-b-0", className)}
      {...props}
    />
  );
}

function AccordionTrigger({
  className,
  children,
  ...props
}: React.ComponentProps<typeof AccordionPrimitive.Trigger>) {
  return (
    <AccordionPrimitive.Header className="flex">
      <AccordionPrimitive.Trigger
        data-slot="accordion-trigger"
        className={cn(
          "flex flex-1 items-center justify-between gap-4 py-4 text-left text-sm font-medium outline-none transition-all hover:underline focus-visible:ring-[3px] focus-visible:ring-ring/50 disabled:pointer-events-none disabled:opacity-50 [&[data-state=open]>svg]:rotate-180",
          className,
        )}
        {...props}
      >
        {children}
        <ChevronDownIcon className="pointer-events-none size-4 shrink-0 text-muted-foreground transition-transform duration-200" />
      </AccordionPrimitive.Trigger>
    </AccordionPrimitive.Header>
  );
}

function AccordionContent({
  className,
  children,
  ...props
}: React.ComponentProps<typeof AccordionPrimitive.Content>) {
  return (
    <AccordionPrimitive.Content
      data-slot="accordion-content"
      className="overflow-hidden text-sm"
      {...props}
    >
      <div className={cn("pb-4 pt-0", className)}>{children}</div>
    </AccordionPrimitive.Content>
  );
}

export { Accordion, AccordionItem, AccordionTrigger, AccordionContent };
```

- [ ] **Step 2: Verify it type-checks / builds**

Run: `npx tsc --noEmit`
Expected: no errors referencing `components/ui/accordion.tsx`.
(If `radix-ui` does not re-export `Accordion`, run `npx tsc --noEmit` to see the named-export error and confirm the exact export — but per the unified package's API it is `Accordion`.)

- [ ] **Step 3: Commit**

```bash
git add components/ui/accordion.tsx
git commit -m "feat(ui): add Accordion primitive (radix-nova)"
```

---

### Task 3: Slider primitive (`components/ui/slider.tsx`)

**Files:**
- Create: `components/ui/slider.tsx`

- [ ] **Step 1: Create the component**

Create `components/ui/slider.tsx`:

```tsx
"use client";

import * as React from "react";
import { Slider as SliderPrimitive } from "radix-ui";

import { cn } from "@/lib/utils";

function Slider({
  className,
  defaultValue,
  value,
  min = 0,
  max = 100,
  ...props
}: React.ComponentProps<typeof SliderPrimitive.Root>) {
  const values = React.useMemo(
    () =>
      Array.isArray(value)
        ? value
        : Array.isArray(defaultValue)
          ? defaultValue
          : [min, max],
    [value, defaultValue, min, max],
  );

  return (
    <SliderPrimitive.Root
      data-slot="slider"
      defaultValue={defaultValue}
      value={value}
      min={min}
      max={max}
      className={cn(
        "relative flex w-full touch-none select-none items-center data-[disabled]:opacity-50",
        className,
      )}
      {...props}
    >
      <SliderPrimitive.Track
        data-slot="slider-track"
        className="relative h-1.5 w-full grow overflow-hidden rounded-full bg-muted"
      >
        <SliderPrimitive.Range data-slot="slider-range" className="absolute h-full bg-primary" />
      </SliderPrimitive.Track>
      {Array.from({ length: values.length }, (_, i) => (
        <SliderPrimitive.Thumb
          data-slot="slider-thumb"
          key={i}
          className="block size-4 shrink-0 rounded-full border border-primary bg-background shadow-sm transition-[color,box-shadow] hover:ring-4 hover:ring-ring/50 focus-visible:outline-hidden focus-visible:ring-4 focus-visible:ring-ring/50 disabled:pointer-events-none"
        />
      ))}
    </SliderPrimitive.Root>
  );
}

export { Slider };
```

- [ ] **Step 2: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: no errors referencing `components/ui/slider.tsx`.

- [ ] **Step 3: Commit**

```bash
git add components/ui/slider.tsx
git commit -m "feat(ui): add Slider primitive (radix-nova)"
```

---

### Task 4: Expose `isDragging()` from the editor hook

**Files:**
- Modify: `components/device-preview/kiosk-editor/use-kiosk-editor.ts`

The carousel must not swipe while an object is being dragged. The hook already tracks an in-progress drag in the `drag` ref; expose a getter.

- [ ] **Step 1: Add `isDragging` to the interface**

In `use-kiosk-editor.ts`, in the `KioskEditor` interface (after `endInteraction: () => void;`), add:

```ts
  /** True while an object move/resize drag is in progress. */
  isDragging: () => boolean;
```

- [ ] **Step 2: Implement and return it**

In `useKioskEditor`, just before the `const ordered = ...` line, add:

```ts
  const isDragging = () => drag.current !== null;
```

Then add `isDragging,` to the returned object (next to `endInteraction,`).

- [ ] **Step 3: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add components/device-preview/kiosk-editor/use-kiosk-editor.ts
git commit -m "feat(branding): expose isDragging() for carousel gesture guard"
```

---

### Task 5: Inline label editing on the canvas

**Files:**
- Modify: `components/device-preview/kiosk-preview.tsx` (confirm `cq` is exported)
- Modify: `components/device-preview/kiosk-editor/kiosk-stage.tsx`

Fixes the "can't edit existing labels" complaint: double-clicking a text object opens an inline `<textarea>` seeded with its text. Enter (or blur) commits via `editor.patch`; Escape cancels.

- [ ] **Step 1: Ensure `cq` is exported from `kiosk-preview.tsx`**

Open `components/device-preview/kiosk-preview.tsx` and find the `cq` helper (the container-query px→cqw function used by the screen renderers). Confirm its declaration is `export function cq(` / `export const cq =`. If it is not exported, add the `export` keyword. (It is already imported by `kiosk-stage.tsx`'s sibling modules via `kioskRootStyle`; this step only guarantees `cq` itself is importable.)

- [ ] **Step 2: Add inline editing to `KioskStage`**

In `components/device-preview/kiosk-editor/kiosk-stage.tsx`:

Update the imports at the top to add `cq` and `MAX_TEXT_LEN`:

```tsx
import { ObjectVisual, kioskRootStyle, cq, type KioskBrand } from "../kiosk-preview";
import { MAX_TEXT_LEN } from "@/lib/kiosk-layout";
```

Inside `KioskStage`, add editing state below the existing `const { ... } = editor;` line:

```tsx
  const [editingId, setEditingId] = React.useState<string | null>(null);

  // Stop editing whenever the canvas unmounts (screen switch) or edit target hides.
  React.useEffect(() => {
    if (editingId && !ordered.some((o) => o.id === editingId && o.visible)) {
      setEditingId(null);
    }
  }, [editingId, ordered]);
```

Replace the object-mapping block (the `{ordered.filter((o) => o.visible).map((o) => ( ... ))}` JSX) with:

```tsx
      {ordered
        .filter((o) => o.visible)
        .map((o) => {
          const editing = editingId === o.id && o.type === "text";
          return (
            <div
              key={o.id}
              onPointerDown={(ev) => {
                if (editing) return; // let the textarea own the pointer
                editor.startMove(o.id, ev);
              }}
              onDoubleClick={() => {
                if (!disabled && o.type === "text") setEditingId(o.id);
              }}
              className={cn("absolute", !disabled && "cursor-grab active:cursor-grabbing")}
              style={{ left: `${o.x * 100}%`, top: `${o.y * 100}%`, width: `${o.w * 100}%`, height: `${o.h * 100}%`, zIndex: editing ? 9998 : o.z }}
            >
              {editing ? (
                <InlineTextEditor
                  object={o}
                  onCommit={(text) => {
                    editor.patch(o.id, { text });
                    setEditingId(null);
                  }}
                  onCancel={() => setEditingId(null)}
                />
              ) : (
                <ObjectVisual object={o} brand={brand} layout={layout} />
              )}
            </div>
          );
        })}
```

- [ ] **Step 3: Add the `InlineTextEditor` component**

At the bottom of `kiosk-stage.tsx` (after `ResizeHandleDot`), add:

```tsx
/** Inline editor shown over a text object on double-click. */
function InlineTextEditor({
  object,
  onCommit,
  onCancel,
}: {
  object: { text?: string; fontSize?: number; align?: "left" | "center" | "right" };
  onCommit: (text: string) => void;
  onCancel: () => void;
}) {
  const ref = React.useRef<HTMLTextAreaElement>(null);
  const [val, setVal] = React.useState(object.text ?? "");

  React.useEffect(() => {
    ref.current?.focus();
    ref.current?.select();
  }, []);

  return (
    <textarea
      ref={ref}
      value={val}
      maxLength={MAX_TEXT_LEN}
      onChange={(e) => setVal(e.target.value)}
      onPointerDown={(e) => e.stopPropagation()}
      onBlur={() => onCommit(val)}
      onKeyDown={(e) => {
        if (e.key === "Enter" && !e.shiftKey) {
          e.preventDefault();
          onCommit(val);
        } else if (e.key === "Escape") {
          e.preventDefault();
          onCancel();
        }
      }}
      className="size-full resize-none rounded-[1cqw] bg-[var(--k-bg)]/90 text-[var(--k-fg)] outline-none ring-2 ring-[var(--k-accent)]"
      style={{
        fontSize: cq(object.fontSize ?? 24),
        textAlign: object.align ?? "center",
        fontWeight: 600,
        lineHeight: 1.1,
        padding: 0,
      }}
    />
  );
}
```

- [ ] **Step 4: Verify build + manual smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: clean build.

Then run `npm run dev`, sign in as `dana@roastwell.co` / `password123`, open **/tenant/branding**, and on the idle preview **double-click the "Lane 1" or tagline text** → an inline editor appears, typing updates it, Enter commits, the save bar shows "Unsaved changes". Escape reverts the in-progress edit.

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/kiosk-preview.tsx components/device-preview/kiosk-editor/kiosk-stage.tsx
git commit -m "feat(branding): double-click to edit text labels inline on the canvas"
```

---

### Task 6: `PreviewCarousel` component

**Files:**
- Create: `components/device-preview/preview-carousel.tsx`

A self-contained, pointer-based swipe carousel. It renders `count` slides via a `renderSlide(i)` prop, constrains each slide's inner content to `slideWidthPx` (the zoom width), and exposes arrows + dots. Swipe is ignored when `isDragging?.()` is true at pointer-down (object drag in progress).

- [ ] **Step 1: Create the component**

Create `components/device-preview/preview-carousel.tsx`:

```tsx
"use client";

import * as React from "react";
import { ChevronLeft, ChevronRight } from "lucide-react";
import { stepIndex, swipeStep } from "@/lib/branding-shell";
import { cn } from "@/lib/utils";

export function PreviewCarousel({
  count,
  index,
  onIndexChange,
  slideWidthPx,
  isDragging,
  renderSlide,
  ariaLabels,
}: {
  count: number;
  index: number;
  onIndexChange: (i: number) => void;
  slideWidthPx: number;
  isDragging?: () => boolean;
  renderSlide: (i: number) => React.ReactNode;
  ariaLabels?: string[];
}) {
  const frameRef = React.useRef<HTMLDivElement>(null);
  const drag = React.useRef<{ startX: number; active: boolean } | null>(null);
  const [dragDx, setDragDx] = React.useState(0);

  function onPointerDown(e: React.PointerEvent) {
    // Don't start a swipe if an object drag is underway, or on a secondary button.
    if (isDragging?.() || e.button !== 0) return;
    drag.current = { startX: e.clientX, active: true };
    setDragDx(0);
  }
  function onPointerMove(e: React.PointerEvent) {
    if (!drag.current?.active) return;
    if (isDragging?.()) {
      // An object grab started after our pointer-down — abandon the swipe.
      drag.current = null;
      setDragDx(0);
      return;
    }
    setDragDx(e.clientX - drag.current.startX);
  }
  function onPointerUp() {
    if (!drag.current?.active) return;
    const width = frameRef.current?.clientWidth ?? 0;
    const dir = swipeStep(dragDx, width);
    if (dir !== 0) onIndexChange(stepIndex(index, dir, count));
    drag.current = null;
    setDragDx(0);
  }

  const go = (dir: number) => onIndexChange(stepIndex(index, dir, count));

  return (
    <div className="space-y-3">
      <div className="relative">
        <div
          ref={frameRef}
          className="overflow-hidden touch-pan-y"
          onPointerDown={onPointerDown}
          onPointerMove={onPointerMove}
          onPointerUp={onPointerUp}
          onPointerLeave={onPointerUp}
        >
          <div
            className={cn("flex", dragDx === 0 && "transition-transform duration-300 ease-out")}
            style={{ transform: `translateX(calc(${-index * 100}% + ${dragDx}px))` }}
          >
            {Array.from({ length: count }, (_, i) => (
              <div key={i} className="w-full shrink-0 px-1">
                <div className="mx-auto" style={{ width: slideWidthPx, maxWidth: "100%" }}>
                  {renderSlide(i)}
                </div>
              </div>
            ))}
          </div>
        </div>

        <CarouselArrow side="left" onClick={() => go(-1)} />
        <CarouselArrow side="right" onClick={() => go(1)} />
      </div>

      <div className="flex items-center justify-center gap-1.5">
        {Array.from({ length: count }, (_, i) => (
          <button
            key={i}
            type="button"
            onClick={() => onIndexChange(i)}
            aria-label={ariaLabels?.[i] ?? `Go to screen ${i + 1}`}
            aria-current={i === index}
            className={cn(
              "h-1.5 rounded-full transition-all",
              i === index ? "w-5 bg-foreground" : "w-1.5 bg-muted-foreground/30 hover:bg-muted-foreground/60",
            )}
          />
        ))}
      </div>
    </div>
  );
}

function CarouselArrow({ side, onClick }: { side: "left" | "right"; onClick: () => void }) {
  const Icon = side === "left" ? ChevronLeft : ChevronRight;
  return (
    <button
      type="button"
      onClick={onClick}
      aria-label={side === "left" ? "Previous screen" : "Next screen"}
      className={cn(
        "absolute top-1/2 z-10 flex size-8 -translate-y-1/2 items-center justify-center rounded-full border bg-background/80 text-muted-foreground shadow-sm backdrop-blur transition-colors hover:text-foreground",
        side === "left" ? "left-1" : "right-1",
      )}
    >
      <Icon className="size-4" />
    </button>
  );
}
```

- [ ] **Step 2: Verify it type-checks**

Run: `npx tsc --noEmit`
Expected: no errors referencing `preview-carousel.tsx`.

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/preview-carousel.tsx
git commit -m "feat(branding): pointer-based preview carousel with arrows + dots"
```

---

### Task 7: Wire zoom + carousel into the preview card

**Files:**
- Modify: `components/branding-editor.tsx`

Replace the static preview card body with the carousel (driven by the existing `screen` state) plus a zoom slider in the header.

- [ ] **Step 1: Add imports**

At the top of `components/branding-editor.tsx`, add to the lucide import list `Minus` and `Plus` if not present, and add these imports below the existing ones:

```tsx
import { Slider } from "@/components/ui/slider";
import { PreviewCarousel } from "@/components/device-preview/preview-carousel";
import { clampZoom, zoomToPx, ZOOM_MIN, ZOOM_MAX, ZOOM_STEP } from "@/lib/branding-shell";
```

- [ ] **Step 2: Add zoom state**

Next to the existing `const [screen, setScreen] = React.useState<KioskScreen>("idle");`, add:

```tsx
  const [zoom, setZoom] = React.useState(80);
  const screenIndex = SCREENS.findIndex((s) => s.value === screen);
  const slidePx = zoomToPx(zoom);
```

- [ ] **Step 3: Replace the preview `Card` (the whole right pane `<Card>…</Card>`)**

Replace the right-pane card (currently lines ~331–358, the `<Card>` containing `CardHeader` with the `Select` and `CardContent` with the conditional `KioskStage`/`KioskPreview`) with:

```tsx
          <Card>
            <CardHeader className="flex-row items-center justify-between space-y-0">
              <div className="space-y-1">
                <CardTitle className="text-base">Live preview</CardTitle>
                <CardDescription>720 × 720 kiosk display</CardDescription>
              </div>
              <Select value={screen} onValueChange={(v) => setScreen(v as KioskScreen)}>
                <SelectTrigger className="w-[150px]" aria-label="Preview screen"><SelectValue /></SelectTrigger>
                <SelectContent>
                  {SCREENS.map((s) => (<SelectItem key={s.value} value={s.value}>{s.label}</SelectItem>))}
                </SelectContent>
              </Select>
            </CardHeader>
            <CardContent className="space-y-4">
              <div className="flex items-center gap-3">
                <span className="text-xs font-medium text-muted-foreground">Zoom</span>
                <button
                  type="button"
                  onClick={() => setZoom((z) => clampZoom(z - ZOOM_STEP))}
                  aria-label="Zoom out"
                  className="flex size-6 items-center justify-center rounded-md border text-muted-foreground transition-colors hover:text-foreground"
                >
                  <Minus className="size-3.5" />
                </button>
                <Slider
                  value={[zoom]}
                  min={ZOOM_MIN}
                  max={ZOOM_MAX}
                  step={ZOOM_STEP}
                  onValueChange={(v) => setZoom(clampZoom(v[0]))}
                  aria-label="Preview zoom"
                  className="flex-1"
                />
                <button
                  type="button"
                  onClick={() => setZoom((z) => clampZoom(z + ZOOM_STEP))}
                  aria-label="Zoom in"
                  className="flex size-6 items-center justify-center rounded-md border text-muted-foreground transition-colors hover:text-foreground"
                >
                  <Plus className="size-3.5" />
                </button>
                <span className="w-10 text-right font-mono text-xs tabular-nums text-muted-foreground">{zoom}%</span>
              </div>

              <PreviewCarousel
                count={SCREENS.length}
                index={screenIndex < 0 ? 0 : screenIndex}
                onIndexChange={(i) => setScreen(SCREENS[i].value)}
                slideWidthPx={slidePx}
                isDragging={editor.isDragging}
                ariaLabels={SCREENS.map((s) => s.label)}
                renderSlide={(i) =>
                  SCREENS[i].value === "idle" ? (
                    <KioskStage editor={editor} brand={kioskBrand} />
                  ) : (
                    <KioskPreview brand={kioskBrand} layout={layout} screen={SCREENS[i].value} />
                  )
                }
              />

              <p className="text-center text-xs text-muted-foreground">
                {screen === "idle"
                  ? "Drag to arrange the idle screen — double-click any text to edit it. Swipe or use the arrows to switch screens."
                  : "Swipe or use the arrows to switch screens. The QR shown is illustrative."}
              </p>
            </CardContent>
          </Card>
```

- [ ] **Step 4: Verify build + manual smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: clean build.

In `npm run dev` on **/tenant/branding**: the zoom slider/`±` buttons resize the preview between 50–125%; swiping the preview (or arrows/dots) cycles through all 7 screens and keeps the dropdown in sync; dragging an idle object does **not** trigger a swipe.

- [ ] **Step 5: Commit**

```bash
git add components/branding-editor.tsx
git commit -m "feat(branding): zoomable, swipeable live preview"
```

---

### Task 8: Replace the left 3-card stack with the accordion

**Files:**
- Modify: `components/branding-editor.tsx`

Collapse Brand / This screen / Security into a single compact accordion. The middle item retitles per screen and (in Phase 1) keeps the existing idle-only controls.

- [ ] **Step 1: Add imports + a per-screen title helper**

Add to the imports in `components/branding-editor.tsx`:

```tsx
import { Accordion, AccordionItem, AccordionTrigger, AccordionContent } from "@/components/ui/accordion";
```

Below the `SCREENS` constant, add:

```tsx
function screenSectionTitle(screen: KioskScreen): string {
  const label = SCREENS.find((s) => s.value === screen)?.label ?? "Screen";
  return screen === "idle" ? "Idle layout" : `${label} content`;
}
```

- [ ] **Step 2: Replace the left pane**

Replace the left-pane container — the `<div className="space-y-6">` that holds the three `<Section …>` blocks (Brand, Idle layout, Security) — with an accordion. Keep the **inner control markup unchanged**; only the wrapper changes from `Section` cards to accordion items:

```tsx
        {/* LEFT — grouped controls */}
        <Card className="self-start">
          <Accordion type="single" collapsible defaultValue="screen" className="px-4">
            <AccordionItem value="brand">
              <AccordionTrigger>
                <SectionHead icon={Palette} title="Brand" />
              </AccordionTrigger>
              <AccordionContent className="space-y-4">
                {/* …existing Brand controls: logo block, logoText input, accent color + presets, advanced theme… */}
              </AccordionContent>
            </AccordionItem>

            <AccordionItem value="screen">
              <AccordionTrigger>
                <SectionHead icon={LayoutGrid} title={screenSectionTitle(screen)} />
              </AccordionTrigger>
              <AccordionContent className="space-y-4">
                {screen === "idle" ? (
                  <KioskControls editor={editor} />
                ) : (
                  <div className="rounded-lg border border-dashed bg-muted/30 p-4 text-center text-sm text-muted-foreground">
                    Switch the preview to{" "}
                    <button type="button" onClick={() => setScreen("idle")} className="font-medium text-foreground underline underline-offset-2">
                      Idle / ready
                    </button>{" "}
                    to edit the layout. Per-screen editing arrives in the next update.
                  </div>
                )}
              </AccordionContent>
            </AccordionItem>

            <AccordionItem value="security">
              <AccordionTrigger>
                <SectionHead icon={ShieldCheck} title="Security" />
              </AccordionTrigger>
              <AccordionContent className="space-y-4">
                {/* …existing Security controls: staff PIN field… */}
              </AccordionContent>
            </AccordionItem>
          </Accordion>
        </Card>
```

Move the existing inner JSX from each old `Section` into the matching `AccordionContent` (Brand → brand item, the Security PIN field → security item; the idle-layout conditional is already shown above). Delete the now-unused `Section` component definition at the bottom of the file.

- [ ] **Step 3: Add the `SectionHead` trigger label**

At the bottom of `components/branding-editor.tsx` (replacing the deleted `Section`), add a compact header used inside accordion triggers:

```tsx
/** Icon + title shown inside an accordion trigger. */
function SectionHead({
  icon: Icon,
  title,
}: {
  icon: React.ComponentType<{ className?: string }>;
  title: string;
}) {
  return (
    <span className="flex items-center gap-2.5">
      <span className="flex size-8 items-center justify-center rounded-lg bg-muted text-muted-foreground">
        <Icon className="size-4" />
      </span>
      <span className="text-base font-semibold">{title}</span>
    </span>
  );
}
```

- [ ] **Step 4: Verify build + manual smoke**

Run: `npx tsc --noEmit && npm run build`
Expected: clean build with no unused-import or unused-`Section` errors (strict mode). If `CardHeader`/`CardTitle`/`CardDescription` become unused after the change, remove them from the import.

In `npm run dev`: the left panel is now a compact accordion (Brand / "<screen> content" / Security); "This screen" is open by default and its title tracks the selected screen; expanding one collapses the others; all Brand, layout, and PIN controls still work and still drive the save bar.

- [ ] **Step 5: Commit**

```bash
git add components/branding-editor.tsx
git commit -m "feat(branding): compact accordion settings panel"
```

---

## Self-Review Notes

- **Spec coverage (Phase 1):** accordion panel → Task 8; zoom slider → Tasks 1, 7; swipe/carousel → Tasks 1, 6, 7; inline label editing → Task 5; gesture guard → Tasks 4, 6. All Phase 1 requirements map to a task.
- **No data-model changes** in this plan — `actions.ts`, `data.ts`, schema untouched, as the spec requires.
- **Type consistency:** `isDragging` (Task 4) is consumed by `PreviewCarousel.isDragging` (Task 6) and passed as `editor.isDragging` (Task 7). `zoomToPx`/`clampZoom`/`ZOOM_*` (Task 1) are consumed in Task 7. `swipeStep`/`stepIndex` (Task 1) are consumed in Task 6. `cq` export (Task 5 Step 1) is consumed by `InlineTextEditor` (Task 5 Step 3).
- **Phase 2** (per-screen object model, `icon` type, migration, editable Success Icon) is intentionally out of this plan and gets its own plan after Phase 1 ships.
```
