# Branding Page Studio Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure `/tenant/branding` into a two-pane studio (grouped controls left, large preview right) with a sticky save bar, by splitting the cramped kiosk editor into a shared hook + a canvas component + a controls component — preserving every existing feature and behavior.

**Architecture:** Extract all kiosk-editor state/handlers from `kiosk-layout-editor.tsx` into a `useKioskEditor` hook. Render the canvas (`KioskStage`) in the right pane and the controls (`KioskControls`) in the left pane, both driven by one hook instance. Rewrite `branding-editor.tsx` as the two-pane shell with grouped sections and a sticky save bar. No logic/geometry/persistence changes — only where things render. The shared `canvasRef`/`elRefs` work across panes because they point at real DOM nodes rendered in the same React commit.

**Tech Stack:** Next.js 16 / React 19 / TypeScript (strict), Tailwind v4 + shadcn (radix-nova), lucide-react, Vitest. No new dependencies.

---

## File structure

- **Create** `components/device-preview/kiosk-editor/use-kiosk-editor.ts` — the `useKioskEditor` hook: owns refs, selection, drag, measurement, and all handlers; returns a `KioskEditor` object consumed by both panes.
- **Create** `components/device-preview/kiosk-editor/kiosk-stage.tsx` — `KioskStage` (the editable canvas + `SelectionOverlay`/`ResizeHandleDot`), right pane.
- **Create** `components/device-preview/kiosk-editor/kiosk-controls.tsx` — `KioskControls` (Add-text + element list + `Inspector`/`NumberField` + Clock/Wi-Fi + Reset layout), left pane.
- **Rewrite** `components/branding-editor.tsx` — two-pane shell: grouped sections (Brand / Idle layout / Security), large sticky preview stage, sticky save bar; instantiates the hook once and wires `KioskControls` + `KioskStage`. Keeps all existing brand state/handlers (`save`/`reset`/`onFile`/`removeLogo`/`commitHex`/`dirty`).
- **Delete** `components/device-preview/kiosk-layout-editor.tsx` — its only consumer is `branding-editor.tsx`; superseded by the three new files.
- **No change:** `lib/kiosk-layout.ts`, `lib/kiosk-geometry.ts` (+ tests), `components/device-preview/kiosk-preview.tsx`, `components/device-preview/kiosk-clock.tsx`, `app/(tenant)/tenant/branding/{page.tsx,actions.ts}`.

**Build note:** Tasks 1–3 add new, not-yet-consumed files; the old `kiosk-layout-editor.tsx` still backs `branding-editor.tsx`, so `tsc` and `vitest` stay green throughout. Task 4 swaps the consumer and deletes the old file. Each task ends with `npx tsc --noEmit` + `npx vitest run` green and a commit.

---

## Task 1: Editor hook (`use-kiosk-editor.ts`)

Lift all editor state + handlers out of the monolithic editor into a reusable hook so the canvas and controls can live in different panes.

**Files:**
- Create: `components/device-preview/kiosk-editor/use-kiosk-editor.ts`

- [ ] **Step 1: Create the hook file**

```ts
"use client";

import * as React from "react";
import {
  DEFAULT_KIOSK_LAYOUT,
  createTextElement,
  MAX_CUSTOM,
  SCALE_MIN,
  SCALE_MAX,
  type KioskElement,
  type KioskLayout,
} from "@/lib/kiosk-layout";
import { resizeBox, clampCenter, type Box, type Handle } from "@/lib/kiosk-geometry";

const clamp = (n: number, lo: number, hi: number) => Math.min(hi, Math.max(lo, n));
const SNAP = 0.025; // snap-to-center threshold (fraction)

type DragKind =
  | { type: "move" }
  | { type: "resize"; handle: Handle; startBox: Box; startEl: KioskElement };

/** Everything the kiosk canvas (KioskStage) and controls (KioskControls) share. */
export interface KioskEditor {
  layout: KioskLayout;
  onChange: (l: KioskLayout) => void;
  disabled: boolean;
  canvasRef: React.RefObject<HTMLDivElement | null>;
  elRefs: React.RefObject<Map<string, HTMLDivElement>>;
  selectedId: string | null;
  setSelectedId: React.Dispatch<React.SetStateAction<string | null>>;
  selected: KioskElement | null;
  guide: { x: boolean; y: boolean };
  selBox: Box | null;
  ordered: KioskElement[];
  atCustomCap: boolean;
  patch: (id: string, p: Partial<KioskElement>) => void;
  startMove: (id: string, e: React.PointerEvent) => void;
  startResize: (handle: Handle, e: React.PointerEvent) => void;
  onPointerMove: (e: React.PointerEvent) => void;
  onPointerUp: (e: React.PointerEvent) => void;
  onCanvasPointerDown: () => void;
  addText: () => void;
  removeEl: (id: string) => void;
  bringToFront: (id: string) => void;
  resetLayout: () => void;
}

/**
 * Owns kiosk idle-layout editing state/handlers so the canvas and the controls
 * can render in separate panes off one instance. `remeasureKey` re-measures the
 * selection overlay when an external visual changes natural size (e.g. the logo).
 */
export function useKioskEditor({
  layout,
  onChange,
  disabled = false,
  remeasureKey,
}: {
  layout: KioskLayout;
  onChange: (l: KioskLayout) => void;
  disabled?: boolean;
  remeasureKey?: unknown;
}): KioskEditor {
  const canvasRef = React.useRef<HTMLDivElement>(null);
  const elRefs = React.useRef<Map<string, HTMLDivElement>>(new Map());
  const drag = React.useRef<DragKind | null>(null);
  const [selectedId, setSelectedId] = React.useState<string | null>(null);
  const [guide, setGuide] = React.useState<{ x: boolean; y: boolean }>({ x: false, y: false });
  const [selBox, setSelBox] = React.useState<Box | null>(null);

  const selected = layout.elements.find((e) => e.id === selectedId) ?? null;
  const customCount = layout.elements.filter((e) => e.kind === "text").length;
  const atCustomCap = customCount >= MAX_CUSTOM;

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
      const sx = d.startBox.w > 0
        ? clamp(d.startEl.sx * (nb.w / d.startBox.w), SCALE_MIN, SCALE_MAX)
        : d.startEl.sx;
      const sy = d.startBox.h > 0
        ? clamp(d.startEl.sy * (nb.h / d.startBox.h), SCALE_MIN, SCALE_MAX)
        : d.startEl.sy;
      patch(selected.id, { x: nb.cx, y: nb.cy, sx, sy });
    }
  }

  function onPointerUp(e: React.PointerEvent) {
    if (drag.current && canvasRef.current?.hasPointerCapture(e.pointerId)) {
      canvasRef.current.releasePointerCapture(e.pointerId);
    }
    drag.current = null;
    setGuide({ x: false, y: false });
  }

  function onCanvasPointerDown() {
    setSelectedId(null);
  }

  function addText() {
    if (disabled || atCustomCap) return;
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

  function resetLayout() {
    onChange(DEFAULT_KIOSK_LAYOUT);
    setSelectedId(null);
  }

  // Keep the handle overlay synced to the selected element's measured visual box.
  React.useLayoutEffect(() => {
    if (!selectedId || !selected?.visible || !elRefs.current.get(selectedId) || !canvasRef.current) {
      setSelBox(null);
      return;
    }
    setSelBox(elementBox(selectedId));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [selectedId, selected?.visible, selected?.x, selected?.y, selected?.sx, selected?.sy, selected?.text, remeasureKey]);

  const ordered = [...layout.elements].sort((a, b) => a.z - b.z);

  return {
    layout,
    onChange,
    disabled,
    canvasRef,
    elRefs,
    selectedId,
    setSelectedId,
    selected,
    guide,
    selBox,
    ordered,
    atCustomCap,
    patch,
    startMove,
    startResize,
    onPointerMove,
    onPointerUp,
    onCanvasPointerDown,
    addText,
    removeEl,
    bringToFront,
    resetLayout,
  };
}
```

- [ ] **Step 2: Type-check + tests**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no errors.
Run: `npx vitest run`
Expected: all pass (148).

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/kiosk-editor/use-kiosk-editor.ts
git commit -m "feat(branding): extract useKioskEditor hook (shared editor state)"
```

---

## Task 2: Canvas component (`kiosk-stage.tsx`)

The editable canvas + selection overlay, consuming the hook. Lives in the right pane.

**Files:**
- Create: `components/device-preview/kiosk-editor/kiosk-stage.tsx`

- [ ] **Step 1: Create the file**

```tsx
"use client";

import * as React from "react";
import { KioskElementView, kioskRootStyle, type KioskBrand } from "../kiosk-preview";
import { HANDLES, type Box, type Handle } from "@/lib/kiosk-geometry";
import type { KioskEditor } from "./use-kiosk-editor";
import { cn } from "@/lib/utils";

/** The editable kiosk canvas: positioned elements, snap guides, and the
 *  selection/resize overlay. Driven entirely by a useKioskEditor instance. */
export function KioskStage({ editor, brand }: { editor: KioskEditor; brand: KioskBrand }) {
  const { layout, disabled, canvasRef, elRefs, ordered, guide, selBox, selectedId } = editor;
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
            onPointerDown={(ev) => editor.startMove(e.id, ev)}
            className={cn("absolute", !disabled && "cursor-grab active:cursor-grabbing")}
            style={{ left: `${e.x * 100}%`, top: `${e.y * 100}%`, transform: "translate(-50%, -50%)", zIndex: e.z }}
          >
            <KioskElementView element={e} brand={brand} layout={layout} />
          </div>
        ))}

      {selBox && selectedId && !disabled && (
        <SelectionOverlay box={selBox} onResizeStart={editor.startResize} />
      )}
    </div>
  );
}

/**
 * Selection ring + 8 resize handles, on its own layer at the element's measured
 * visual box. Pointer-transparent except the handle dots, so dragging the body
 * underneath still moves it.
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
```

- [ ] **Step 2: Type-check + tests**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no errors.
Run: `npx vitest run`
Expected: all pass (148).

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/kiosk-editor/kiosk-stage.tsx
git commit -m "feat(branding): KioskStage canvas component (right pane)"
```

---

## Task 3: Controls component (`kiosk-controls.tsx`)

Add-text + element list + inspector + clock/wifi + reset, consuming the hook. Lives in the left pane.

**Files:**
- Create: `components/device-preview/kiosk-editor/kiosk-controls.tsx`

- [ ] **Step 1: Create the file**

```tsx
"use client";

import * as React from "react";
import { Eye, EyeOff, Plus, RotateCcw, Trash2, Wifi } from "lucide-react";
import {
  elementLabel,
  MAX_CUSTOM,
  MAX_TEXT_LEN,
  SCALE_MIN,
  SCALE_MAX,
  type KioskElement,
} from "@/lib/kiosk-layout";
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
const CANVAS_REF_PX = 720; // design reference for pixel readouts

/** All idle-layout controls (element list, inspector, clock/wifi, reset). */
export function KioskControls({ editor }: { editor: KioskEditor }) {
  const { layout, onChange, disabled, ordered, selectedId, setSelectedId, selected, atCustomCap } = editor;

  return (
    <div className="space-y-4">
      {/* Add / element list */}
      <div className="space-y-2 rounded-xl border p-3">
        <div className="flex items-center justify-between">
          <span className="text-sm font-medium">Elements</span>
          <button
            type="button"
            disabled={disabled || atCustomCap}
            onClick={editor.addText}
            title={atCustomCap ? `Limit of ${MAX_CUSTOM} custom text elements reached` : undefined}
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
                onClick={(ev) => { ev.stopPropagation(); editor.patch(e.id, { visible: !e.visible }); }}
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
                  onClick={(ev) => { ev.stopPropagation(); editor.removeEl(e.id); }}
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

      {/* Inspector for the selected element */}
      {selected && <Inspector key={selected.id} el={selected} editor={editor} />}

      {/* Clock + Wi-Fi */}
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
        onClick={editor.resetLayout}
        className="flex w-full items-center justify-center gap-2 rounded-lg border py-2 text-sm font-medium text-muted-foreground transition-colors hover:bg-accent hover:text-foreground disabled:opacity-50"
      >
        <RotateCcw className="size-4" /> Reset layout to default
      </button>
    </div>
  );
}

/** Inspector: pixel X/Y/W/H number boxes + per-kind controls for one element. */
function Inspector({ el, editor }: { el: KioskElement; editor: KioskEditor }) {
  const { canvasRef, elRefs, disabled } = editor;
  const onPatch = (p: Partial<KioskElement>) => editor.patch(el.id, p);

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
        <button type="button" disabled={disabled} onClick={() => editor.bringToFront(el.id)} className="text-xs text-muted-foreground hover:text-foreground disabled:opacity-50">
          Bring to front
        </button>
      </div>

      {el.kind === "text" && (
        <div className="space-y-1.5">
          <Label className="text-xs text-muted-foreground">Text</Label>
          <Input value={el.text ?? ""} disabled={disabled} maxLength={MAX_TEXT_LEN} onChange={(e) => onPatch({ text: e.target.value })} className="h-8" />
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

/** Small labelled numeric input (px). Buffers keystrokes locally and commits on
 *  blur/Enter so multi-digit typing isn't fought by parent re-renders; while not
 *  focused it tracks the live `value` (e.g. during a drag). */
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
  const [draft, setDraft] = React.useState<string>(value?.toString() ?? "");
  const [focused, setFocused] = React.useState(false);

  React.useEffect(() => {
    if (!focused) setDraft(value?.toString() ?? "");
  }, [value, focused]);

  const commit = () => {
    const n = Number(draft);
    if (draft.trim() !== "" && Number.isFinite(n)) onChange(n);
    else setDraft(value?.toString() ?? "");
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
        onKeyDown={(e) => {
          if (e.key === "Enter") {
            commit();
            (e.currentTarget as HTMLInputElement).blur();
          }
        }}
        className="h-8 px-2 font-mono text-xs tabular-nums"
      />
    </div>
  );
}
```

- [ ] **Step 2: Type-check + tests**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no errors.
Run: `npx vitest run`
Expected: all pass (148).

- [ ] **Step 3: Commit**

```bash
git add components/device-preview/kiosk-editor/kiosk-controls.tsx
git commit -m "feat(branding): KioskControls component (left pane)"
```

---

## Task 4: Two-pane shell (`branding-editor.tsx`) + delete old editor

Rewrite the page into a two-pane studio with grouped sections and a sticky save bar, wiring the hook + stage + controls. Then delete the superseded monolithic editor.

**Files:**
- Rewrite: `components/branding-editor.tsx`
- Delete: `components/device-preview/kiosk-layout-editor.tsx`

- [ ] **Step 1: Replace the entire contents of `components/branding-editor.tsx`**

```tsx
"use client";

import * as React from "react";
import { useRouter } from "next/navigation";
import {
  Eye,
  EyeOff,
  ImageUp,
  LayoutGrid,
  Loader2,
  Lock,
  Palette,
  RotateCcw,
  Save,
  ShieldCheck,
  X,
} from "lucide-react";
import { toast } from "sonner";
import {
  KioskPreview,
  type KioskScreen,
  type KioskBrand,
} from "@/components/device-preview/kiosk-preview";
import { useKioskEditor } from "@/components/device-preview/kiosk-editor/use-kiosk-editor";
import { KioskStage } from "@/components/device-preview/kiosk-editor/kiosk-stage";
import { KioskControls } from "@/components/device-preview/kiosk-editor/kiosk-controls";
import { type KioskLayout } from "@/lib/kiosk-layout";
import { Button } from "@/components/ui/button";
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { saveBranding } from "@/app/(tenant)/tenant/branding/actions";
import { isValidHex } from "@/lib/color";
import { cn } from "@/lib/utils";

const PRESETS = ["#B4541F", "#3F9D4E", "#1F5C8B", "#E5484D", "#7C5CFC", "#0F766E", "#111827"];

const SCREENS: { value: KioskScreen; label: string }[] = [
  { value: "idle", label: "Idle / ready" },
  { value: "processing", label: "Processing" },
  { value: "qr", label: "Receipt ready" },
  { value: "sent", label: "Sent ✓" },
  { value: "error", label: "Error / offline" },
  { value: "paused", label: "Paused" },
  { value: "setup", label: "Setup / pairing" },
];

export function BrandingEditor({
  initialColor,
  initialLayout,
  initialBg,
  initialFg,
  initialMuted,
  initialLogoText,
  initialLogoUrl,
  initialStaffPin,
  storeName,
  canEdit,
}: {
  initialColor: string;
  initialLayout: KioskLayout;
  initialBg: string;
  initialFg: string;
  initialMuted: string;
  initialLogoText: string;
  initialLogoUrl: string | null;
  initialStaffPin: string;
  storeName: string;
  canEdit: boolean;
}) {
  const router = useRouter();
  const [color, setColor] = React.useState(initialColor);
  const [hexInput, setHexInput] = React.useState(initialColor);
  const [bg, setBg] = React.useState(initialBg);
  const [fg, setFg] = React.useState(initialFg);
  const [muted, setMuted] = React.useState(initialMuted);
  const [layout, setLayout] = React.useState<KioskLayout>(initialLayout);
  const [logoText, setLogoText] = React.useState(initialLogoText);
  const [logoPreview, setLogoPreview] = React.useState<string | null>(initialLogoUrl);
  const [logoFile, setLogoFile] = React.useState<File | null>(null);
  const [logoCleared, setLogoCleared] = React.useState(false);
  const [pin, setPin] = React.useState(initialStaffPin);
  const [showPin, setShowPin] = React.useState(false);
  const [screen, setScreen] = React.useState<KioskScreen>("idle");
  const [saving, setSaving] = React.useState(false);
  const fileRef = React.useRef<HTMLInputElement>(null);

  const editor = useKioskEditor({
    layout,
    onChange: setLayout,
    disabled: !canEdit,
    remeasureKey: logoPreview,
  });

  const dirty =
    color !== initialColor ||
    bg !== initialBg ||
    fg !== initialFg ||
    muted !== initialMuted ||
    JSON.stringify(layout) !== JSON.stringify(initialLayout) ||
    pin !== initialStaffPin ||
    logoFile !== null ||
    logoCleared;

  function commitHex(v: string) {
    setHexInput(v);
    if (isValidHex(v)) setColor(v.startsWith("#") ? v : `#${v}`);
  }

  function onFile(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;
    if (!file.type.startsWith("image/")) {
      toast.error("Logo must be an image file.");
      return;
    }
    if (file.size > 2 * 1024 * 1024) {
      toast.error("Logo must be under 2 MB.");
      return;
    }
    setLogoFile(file);
    setLogoCleared(false);
    setLogoPreview(URL.createObjectURL(file));
  }

  function removeLogo() {
    setLogoFile(null);
    setLogoPreview(null);
    setLogoCleared(true);
    if (fileRef.current) fileRef.current.value = "";
  }

  function reset() {
    setColor(initialColor);
    setHexInput(initialColor);
    setBg(initialBg);
    setFg(initialFg);
    setMuted(initialMuted);
    setLayout(initialLayout);
    setLogoText(initialLogoText);
    setLogoPreview(initialLogoUrl);
    setLogoFile(null);
    setLogoCleared(false);
    setPin(initialStaffPin);
    if (fileRef.current) fileRef.current.value = "";
  }

  async function save() {
    if (!isValidHex(hexInput)) {
      toast.error("Enter a valid hex color first.");
      return;
    }
    setSaving(true);
    const fd = new FormData();
    fd.set("brandColor", color);
    fd.set("brandBg", bg);
    fd.set("brandFg", fg);
    fd.set("brandMuted", muted);
    fd.set("kioskLayout", JSON.stringify(layout));
    fd.set("staffPin", pin);
    if (logoFile) fd.set("logo", logoFile);
    fd.set("removeLogo", logoCleared ? "true" : "false");

    const res = await saveBranding(fd);
    setSaving(false);

    if (!res.ok) {
      toast.error("Couldn't save branding", { description: res.error });
      return;
    }
    toast.success("Branding saved", { description: "Your kiosks will update on next sync." });
    setLogoFile(null);
    setLogoCleared(false);
    router.refresh();
  }

  const disabled = !canEdit || saving;
  const kioskBrand: KioskBrand = {
    brandColor: color,
    brandBg: bg,
    brandFg: fg,
    brandMuted: muted,
    logoText,
    logoUrl: logoPreview,
    storeName,
  };

  return (
    <div className="relative space-y-6">
      {!canEdit && (
        <div className="flex items-center gap-2 rounded-lg border bg-muted/40 p-3 text-sm text-muted-foreground">
          <Lock className="size-4 shrink-0" />
          You have view-only access. Only owners and admins can edit branding.
        </div>
      )}

      <div className="grid gap-6 lg:grid-cols-[minmax(0,1fr)_minmax(0,1.05fr)] lg:items-start">
        {/* LEFT — grouped controls */}
        <div className="space-y-6">
          <Section icon={Palette} title="Brand" description="Your logo and colors, shown to customers on the kiosk.">
            {logoPreview ? (
              <div className="flex items-center gap-3 rounded-xl border bg-muted/30 p-3">
                {/* eslint-disable-next-line @next/next/no-img-element */}
                <img src={logoPreview} alt="Logo preview" className="size-12 rounded-lg object-contain" />
                <div className="flex flex-1 gap-2">
                  <Button type="button" variant="outline" size="sm" disabled={disabled} onClick={() => fileRef.current?.click()}>
                    <ImageUp className="size-4" /> Replace
                  </Button>
                  <Button type="button" variant="ghost" size="sm" disabled={disabled} onClick={removeLogo}>
                    <X className="size-4" /> Remove
                  </Button>
                </div>
              </div>
            ) : (
              <button
                type="button"
                disabled={disabled}
                onClick={() => fileRef.current?.click()}
                className="flex w-full flex-col items-center justify-center gap-2 rounded-xl border border-dashed bg-muted/30 px-4 py-8 text-center transition-colors hover:bg-muted/60 disabled:cursor-not-allowed disabled:opacity-60"
              >
                <span className="flex size-10 items-center justify-center rounded-lg bg-background text-muted-foreground"><ImageUp className="size-5" /></span>
                <span className="text-sm font-medium">Click to upload a logo</span>
                <span className="text-xs text-muted-foreground">PNG or SVG, transparent background recommended</span>
              </button>
            )}
            <input ref={fileRef} type="file" accept="image/*" className="hidden" onChange={onFile} disabled={disabled} />

            <div className="space-y-2">
              <Label htmlFor="logoText">Logo text (preview fallback)</Label>
              <Input id="logoText" value={logoText} onChange={(e) => setLogoText(e.target.value)} placeholder="Your brand" disabled={disabled} />
            </div>

            <div className="space-y-3">
              <div className="flex items-center gap-3">
                <label className="relative size-12 shrink-0 cursor-pointer overflow-hidden rounded-xl ring-1 ring-border" style={{ background: color }}>
                  <input
                    type="color"
                    value={color}
                    onChange={(e) => { setColor(e.target.value); setHexInput(e.target.value); }}
                    disabled={disabled}
                    className="absolute inset-0 cursor-pointer opacity-0"
                    aria-label="Pick accent color"
                  />
                </label>
                <div className="flex-1 space-y-1.5">
                  <Label htmlFor="hex">Accent color (hex)</Label>
                  <Input id="hex" value={hexInput} onChange={(e) => commitHex(e.target.value)} disabled={disabled} className="font-mono" aria-invalid={!isValidHex(hexInput)} />
                </div>
              </div>
              <div className="flex flex-wrap gap-2">
                {PRESETS.map((c) => (
                  <button
                    key={c}
                    type="button"
                    disabled={disabled}
                    onClick={() => { setColor(c); setHexInput(c); }}
                    className={cn(
                      "size-8 rounded-lg ring-1 ring-border transition-transform hover:scale-110 disabled:cursor-not-allowed disabled:opacity-60",
                      color.toLowerCase() === c.toLowerCase() && "ring-2 ring-foreground ring-offset-2 ring-offset-background",
                    )}
                    style={{ background: c }}
                    aria-label={`Use ${c}`}
                  />
                ))}
              </div>
            </div>

            <div className="space-y-3 rounded-lg border bg-muted/20 p-3">
              <p className="text-xs font-medium text-muted-foreground">Advanced theme — leave as-is for the default look</p>
              <ColorField label="Background" value={bg} onChange={setBg} disabled={disabled} />
              <ColorField label="Text" value={fg} onChange={setFg} disabled={disabled} />
              <ColorField label="Muted text" value={muted} onChange={setMuted} disabled={disabled} />
            </div>
          </Section>

          <Section icon={LayoutGrid} title="Idle layout" description="Arrange what customers see on the idle screen.">
            {screen === "idle" ? (
              <KioskControls editor={editor} />
            ) : (
              <div className="rounded-lg border border-dashed bg-muted/30 p-4 text-center text-sm text-muted-foreground">
                Switch the preview to{" "}
                <button type="button" onClick={() => setScreen("idle")} className="font-medium text-foreground underline underline-offset-2">
                  Idle / ready
                </button>{" "}
                to edit the layout.
              </div>
            )}
          </Section>

          <Section icon={ShieldCheck} title="Security" description="Unlocks on-device settings at the kiosk.">
            <div className="space-y-2">
              <Label htmlFor="staffPin">Staff PIN</Label>
              <div className="relative">
                <Input
                  id="staffPin"
                  value={pin}
                  onChange={(e) => setPin(e.target.value.replace(/\D/g, "").slice(0, 6))}
                  type={showPin ? "text" : "password"}
                  inputMode="numeric"
                  disabled={disabled}
                  className="pr-10 font-mono tracking-[0.3em]"
                  placeholder="••••"
                />
                <button
                  type="button"
                  onClick={() => setShowPin((s) => !s)}
                  className="absolute right-2 top-1/2 -translate-y-1/2 text-muted-foreground hover:text-foreground"
                  aria-label={showPin ? "Hide PIN" : "Show PIN"}
                >
                  {showPin ? <EyeOff className="size-4" /> : <Eye className="size-4" />}
                </button>
              </div>
            </div>
          </Section>
        </div>

        {/* RIGHT — stage */}
        <div className="lg:sticky lg:top-24 lg:self-start">
          <Card>
            <CardHeader className="flex-row items-center justify-between space-y-0">
              <div className="space-y-1">
                <CardTitle className="text-base">Live preview</CardTitle>
                <CardDescription>720 × 720 kiosk display</CardDescription>
              </div>
              <Select value={screen} onValueChange={(v) => setScreen(v as KioskScreen)}>
                <SelectTrigger className="w-[170px]" aria-label="Preview screen"><SelectValue /></SelectTrigger>
                <SelectContent>
                  {SCREENS.map((s) => (<SelectItem key={s.value} value={s.value}>{s.label}</SelectItem>))}
                </SelectContent>
              </Select>
            </CardHeader>
            <CardContent>
              <div className="mx-auto w-full max-w-[600px]">
                {screen === "idle" ? (
                  <KioskStage editor={editor} brand={kioskBrand} />
                ) : (
                  <KioskPreview brand={kioskBrand} layout={layout} screen={screen} />
                )}
              </div>
              <p className="mt-4 text-center text-xs text-muted-foreground">
                {screen === "idle"
                  ? "Drag to arrange the idle screen. Other screens preview your theme."
                  : "The QR code shown is illustrative. Real kiosks render a scannable receipt code."}
              </p>
            </CardContent>
          </Card>
        </div>
      </div>

      {/* STICKY SAVE BAR */}
      {canEdit && (
        <div className="sticky bottom-4 z-30 mt-2 flex items-center justify-between gap-3 rounded-xl border bg-background/85 px-4 py-3 shadow-lg backdrop-blur supports-[backdrop-filter]:bg-background/70">
          <span className="flex items-center gap-2 text-sm text-muted-foreground">
            <span className={cn("size-2 rounded-full", dirty ? "bg-amber-500" : "bg-emerald-500")} />
            {dirty ? "Unsaved changes" : "All changes saved"}
          </span>
          <div className="flex gap-2">
            <Button variant="outline" onClick={reset} disabled={disabled || !dirty}>
              <RotateCcw className="size-4" /> Reset
            </Button>
            <Button onClick={save} disabled={disabled || !dirty}>
              {saving ? <Loader2 className="size-4 animate-spin" /> : <Save className="size-4" />}
              {saving ? "Saving…" : "Save branding"}
            </Button>
          </div>
        </div>
      )}
    </div>
  );
}

/** A titled section group with an icon header. */
function Section({
  icon: Icon,
  title,
  description,
  children,
}: {
  icon: React.ComponentType<{ className?: string }>;
  title: string;
  description: string;
  children: React.ReactNode;
}) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-center gap-2.5">
          <span className="flex size-8 items-center justify-center rounded-lg bg-muted text-muted-foreground">
            <Icon className="size-4" />
          </span>
          <div className="space-y-0.5">
            <CardTitle className="text-base">{title}</CardTitle>
            <CardDescription>{description}</CardDescription>
          </div>
        </div>
      </CardHeader>
      <CardContent className="space-y-4">{children}</CardContent>
    </Card>
  );
}

/** Swatch + hex input for a single theme token. */
function ColorField({
  label,
  value,
  onChange,
  disabled,
}: {
  label: string;
  value: string;
  onChange: (v: string) => void;
  disabled?: boolean;
}) {
  const [hex, setHex] = React.useState(value);
  React.useEffect(() => setHex(value), [value]);
  return (
    <div className="flex items-center gap-3">
      <label className="relative size-9 shrink-0 cursor-pointer overflow-hidden rounded-lg ring-1 ring-border" style={{ background: value }}>
        <input
          type="color"
          value={value}
          onChange={(e) => onChange(e.target.value)}
          disabled={disabled}
          className="absolute inset-0 cursor-pointer opacity-0"
          aria-label={`Pick ${label}`}
        />
      </label>
      <div className="flex-1 space-y-1">
        <Label className="text-xs text-muted-foreground">{label}</Label>
        <Input
          value={hex}
          onChange={(e) => {
            setHex(e.target.value);
            if (isValidHex(e.target.value)) {
              onChange(e.target.value.startsWith("#") ? e.target.value : `#${e.target.value}`);
            }
          }}
          disabled={disabled}
          className="h-8 font-mono text-xs"
          aria-invalid={!isValidHex(hex)}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Delete the superseded monolithic editor**

```bash
git rm components/device-preview/kiosk-layout-editor.tsx
```

- [ ] **Step 3: Type-check the whole project**

Run: `npx tsc --noEmit -p tsconfig.json`
Expected: no errors. (If anything still imports `kiosk-layout-editor`, it's a leftover — there should be none; `branding-editor.tsx` is its only consumer and now uses the new components.)

- [ ] **Step 4: Run the full unit suite**

Run: `npx vitest run`
Expected: all pass (148).

- [ ] **Step 5: Commit**

```bash
git add components/branding-editor.tsx components/device-preview/kiosk-layout-editor.tsx
git commit -m "feat(branding): two-pane studio layout + sticky save bar; split kiosk editor"
```

---

## Task 5: Build + verification

Confirm the whole app builds and the redesigned page behaves. (Interactive browser checks run against production, where auth works — see the spec's Testing section; do them after the next deploy, without saving on the demo tenant except one deliberate round-trip.)

**Files:** none (verification only). Fix any defect in the owning file from Tasks 1–4 and re-run.

- [ ] **Step 1: Production build**

Run: `npm run build`
Expected: build succeeds with no type errors.

- [ ] **Step 2: Confirm the old editor is fully removed and unreferenced**

Run: `grep -rn "kiosk-layout-editor\|KioskLayoutEditor" components app lib || echo "no references — clean"`
Expected: `no references — clean`.

- [ ] **Step 3: Live verification (post-deploy, production)**

On the deployed site, sign in (`dana@roastwell.co` / `123456`) and open `/tenant/branding`. Verify, WITHOUT clicking Save (to avoid mutating the demo tenant) except one deliberate round-trip at the end:
- Two-pane layout: grouped sections (Brand / Idle layout / Security) on the left, large preview on the right; sticky save bar at the bottom showing "All changes saved".
- Brand: logo upload shows the dashed dropzone; once a logo exists, the preview chip with Replace/Remove shows; accent color + presets + the "Advanced theme" subgroup work.
- Idle layout section shows the editor controls (Add text / element list / inspector / clock-wifi / reset) on the LEFT, with the canvas + drag handles on the RIGHT; select / drag / corner-resize / edge-stretch / inspector typing (multi-digit) / add-text / delete-custom all work.
- Switch the preview to a non-idle screen → the Idle layout section shows the "Switch the preview to Idle…" hint; switching back restores the controls.
- Make an edit → save bar flips to "● Unsaved changes"; Save persists (one deliberate save), reload confirms persistence; Reset reverts.
- View-only account (member role) sees the notice and disabled controls, no save bar.

- [ ] **Step 4: Commit any fixes** (skip if none)

```bash
git add -A
git commit -m "fix(branding): address issues found in verification"
```

---

## Self-review notes (coverage map)

- Spec §1 (shell + save flow): Task 4 (two-pane grid, sticky save bar with dirty indicator, responsive `lg` stacking, view-only notice).
- Spec §2 (left grouped controls): Task 4 `Section` groups — Brand (logo chip + text + accent + Advanced theme subgroup), Idle layout (KioskControls or hint), Security (PIN).
- Spec §3 (stage): Task 4 right pane (larger `max-w-[600px]`, sticky, screen selector, KioskStage for idle else KioskPreview).
- Spec §4 (editor refactor): Task 1 `useKioskEditor`, Task 2 `KioskStage`, Task 3 `KioskControls`, Task 4 wiring + delete old editor.
- Spec Testing: Tasks 1–4 keep `tsc`/vitest green; Task 5 build + grep + live verification.
- Type consistency: `KioskEditor` interface (Task 1) is consumed by `KioskStage` (Task 2: `{ editor, brand }`), `KioskControls`/`Inspector` (Task 3: `{ editor }` / `{ el, editor }`), and instantiated once in `branding-editor.tsx` (Task 4) via `useKioskEditor({ layout, onChange, disabled, remeasureKey })`. `KioskBrand` imported from `kiosk-preview` in Tasks 2 + 4. No behavior/logic change to geometry, normalize, or persistence.
```
