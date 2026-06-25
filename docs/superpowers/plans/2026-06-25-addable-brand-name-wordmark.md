# Addable Brand-Name Wordmark Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the brand-name wordmark (the `logo` widget) an addable + deletable per-screen singleton in the layout editor, so any org — including those whose logo was migrated to an image — can place it.

**Architecture:** Admin/editor-only. The `logo` wire type, cloud payload, preview renderer, and firmware are all unchanged (the device already renders `logo` as the org wordmark, HIL-verified 2026-06-25). We relabel the `logo` widget to "Brand name", add a singleton-guarded "+ Brand name" editor button + an `addBrandName()` action, and make the widget deletable.

**Tech Stack:** Next.js 16 / React 19 / TypeScript, Vitest. Files: `lib/printer-layout.ts`, `components/device-preview/printer-editor/use-printer-editor.ts`, `components/device-preview/printer-editor/printer-controls.tsx`.

## Global Constraints

- **Singleton:** at most one `logo` (brand-name) widget per screen. The "+ Brand name" button shows ONLY when the active screen has no `logo` object.
- **Wire type stays `logo`** — do NOT add a new object type, do NOT touch `OBJECT_TYPES`/`WIDGET_TYPES`/`ADDABLE_TYPES`/`sanitizeScreen`/the cloud payload/the firmware. Only the UI label changes.
- **Does NOT count against `MAX_CUSTOM` (20).** It's a widget singleton, not a custom text/icon/image.
- **Deletable + hideable + movable.** Hide (eye toggle) and drag-move already work; this plan adds add + delete.
- **Default box** for a new brand-name widget: `{ x: 0.25, y: 0.32, w: 0.5, h: 0.16 }`, `id: "logo"`, `visible: true`.
- Tests: `npm run test`. Build/type-check: `npm run build` / `npx tsc --noEmit`.
- Branch: `feat/addable-brand-name` (already created).
- Commit footer: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`

---

## File Structure

- `lib/printer-layout.ts` — relabel `logo` → "Brand name"; add `createBrandNameObject(z)`. (Task 1)
- `lib/printer-layout.test.ts` — factory + label + ≤1-singleton tests. (Task 1)
- `components/device-preview/printer-editor/use-printer-editor.ts` — `hasBrandName` + `addBrandName()`. (Task 2)
- `components/device-preview/printer-editor/printer-controls.tsx` — "+ Brand name" button (singleton-guarded) + make `logo` deletable. (Task 2)
- Prod DB cleanup — remove the 2026-06-25 HIL test injection. (Task 3)

---

## Task 1: Relabel + factory (lib/printer-layout.ts)

**Files:**
- Modify: `lib/printer-layout.ts` (`TYPE_LABEL.logo` at line 34; add factory after `createImageObject` ~line 196)
- Test: `lib/printer-layout.test.ts`

**Interfaces:**
- Produces: `createBrandNameObject(z: number): PrinterObject` (a `logo`-type object); `TYPE_LABEL.logo === "Brand name"`.
- Consumes: existing `PrinterObject`, `genIdSuffix` is NOT used (id is fixed `"logo"`), `normalizePrinterConfig`.

- [ ] **Step 1: Write the failing tests**

Append to `lib/printer-layout.test.ts`:

```ts
import { createBrandNameObject, TYPE_LABEL, normalizePrinterConfig } from "./printer-layout";

describe("brand-name (logo) widget", () => {
  it("createBrandNameObject returns a logo object with the default box", () => {
    const o = createBrandNameObject(7);
    expect(o.type).toBe("logo");
    expect(o.id).toBe("logo");
    expect(o.visible).toBe(true);
    expect(o.z).toBe(7);
    expect({ x: o.x, y: o.y, w: o.w, h: o.h }).toEqual({ x: 0.25, y: 0.32, w: 0.5, h: 0.16 });
  });

  it("is labelled 'Brand name'", () => {
    expect(TYPE_LABEL.logo).toBe("Brand name");
  });

  it("normalize keeps at most one logo widget per screen", () => {
    const cfg = normalizePrinterConfig({
      version: 3, clockTimezone: "UTC", clock24h: false, wifiLevel: 3, qrTimeoutSeconds: 60,
      screens: { idle: { objects: [
        { id: "logo", type: "logo", x: 0.25, y: 0.32, w: 0.5, h: 0.16, visible: true, z: 0 },
        { id: "logo2", type: "logo", x: 0.1, y: 0.1, w: 0.2, h: 0.1, visible: true, z: 1 },
      ] } },
    });
    expect(cfg.screens.idle.objects.filter((o) => o.type === "logo").length).toBe(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm run test -- printer-layout`
Expected: FAIL — `createBrandNameObject` not exported / label still "Logo".

- [ ] **Step 3: Implement**

In `lib/printer-layout.ts`, change line 34 in `TYPE_LABEL`:
```ts
  logo: "Brand name",
```
Add a factory after `createImageObject` (after ~line 205, just after that function's closing brace):
```ts
/** A fresh brand-name (logo) widget that renders the org wordmark. Singleton per screen. */
export function createBrandNameObject(z: number): PrinterObject {
  return {
    id: "logo",
    type: "logo",
    x: 0.25, y: 0.32, w: 0.5, h: 0.16,
    visible: true,
    z,
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm run test -- printer-layout`
Expected: PASS (new tests green; existing tests still green — note: any existing test asserting `TYPE_LABEL.logo === "Logo"` must be updated to "Brand name"; search the test file and fix if present).

- [ ] **Step 5: Commit**

```bash
git add lib/printer-layout.ts lib/printer-layout.test.ts
git commit -m "feat(branding): relabel logo widget to Brand name + add factory

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Editor — add/delete the brand-name widget

**Files:**
- Modify: `components/device-preview/printer-editor/use-printer-editor.ts`
- Modify: `components/device-preview/printer-editor/printer-controls.tsx`

**Interfaces:**
- Consumes: `createBrandNameObject` (Task 1).
- Produces: `editor.hasBrandName: boolean`, `editor.addBrandName(): void`.

- [ ] **Step 1: Add `hasBrandName` + `addBrandName` to the hook**

In `use-printer-editor.ts`:
- Import `createBrandNameObject` (add to the existing `@/lib/printer-layout` import alongside `createImageObject`).
- In the `PrinterEditor` interface, after `addImage: () => void;` (line 48):
```ts
  addBrandName: () => void;
  hasBrandName: boolean;
```
- After the `addableCount`/`atCustomCap` lines (~line 80-81), add:
```ts
  const hasBrandName = objects.some((o) => o.type === "logo");
```
- After `addImage()` (after ~line 172, its closing brace), add:
```ts
  function addBrandName() {
    if (disabled || hasBrandName) return;
    const z = objects.reduce((m, o) => Math.max(m, o.z), 0) + 1;
    const o = createBrandNameObject(z);
    setObjects([...objects, o]);
    setSelectedId(o.id);
  }
```
- In the returned object (after `addImage,` ~line 227), add:
```ts
    addBrandName,
    hasBrandName,
```

- [ ] **Step 2: Build to verify the hook compiles**

Run: `npx tsc --noEmit`
Expected: no new errors. (The button consuming these comes next.)

- [ ] **Step 3: Add the "+ Brand name" button (singleton-guarded) + make logo deletable**

In `printer-controls.tsx`:
- Add the button after the "Add image" button (after the `</button>` that closes the Add-image button, ~line 69), inside the same `<div className="flex gap-1">`:
```tsx
            {!editor.hasBrandName && (
              <button
                type="button"
                disabled={disabled}
                onClick={editor.addBrandName}
                className="flex items-center gap-1 rounded-md border px-2 py-1 text-xs font-medium transition-colors hover:bg-accent disabled:opacity-50"
              >
                <Plus className="size-3.5" /> Add brand name
              </button>
            )}
```
- Make the brand-name widget deletable — update the list delete-button condition (line 92) from:
```tsx
              {(o.type === "text" || o.type === "icon" || o.type === "image") && (
```
to:
```tsx
              {(o.type === "text" || o.type === "icon" || o.type === "image" || o.type === "logo") && (
```

- [ ] **Step 4: Build + manual smoke**

Run: `npm run build`
Expected: compiles clean.
Manual (`npm run dev`, sign in `dana@roastwell.co` / `123456`, `/tenant/branding`):
- On a screen with no brand name (e.g. Roastwell idle after the Task 3 cleanup, or hide/delete any existing one): the **"+ Brand name"** button appears; click it → a "Brand name" object is added showing the org name in the preview; the button disappears (singleton).
- The object list shows a trash icon for "Brand name"; deleting it removes the widget and the "+ Brand name" button returns.
- Save; reload; the change persists.

- [ ] **Step 5: Commit**

```bash
git add components/device-preview/printer-editor/use-printer-editor.ts components/device-preview/printer-editor/printer-controls.tsx
git commit -m "feat(branding): add + delete the brand-name widget in the editor

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Remove the HIL test injection (prod DB)

**Files:** none (one-off prod data cleanup).

**Context:** During the 2026-06-25 HIL session a `logo` widget was injected into Roastwell Coffee's idle screen to verify `render_wordmark`. Remove it so Roastwell starts clean and the new "+ Brand name" button can be dogfooded. `.env.local`'s `DATABASE_URL` IS prod — this runs against production.

- [ ] **Step 1: Remove the injected logo widget**

Create `lib/db/_cleanup_hil_logo.ts`:
```ts
import "./load-env";
import { db } from "../db";
import { tenantSettings, organization } from "./schema";
import { eq } from "drizzle-orm";
import { normalizePrinterConfig } from "../printer-layout";
import { enqueueConfigChangedForOrg } from "../data";
async function main() {
  const [org] = await db.select().from(organization).where(eq(organization.name, "Roastwell Coffee")).limit(1);
  const [s] = await db.select().from(tenantSettings).where(eq(tenantSettings.organizationId, org.id)).limit(1);
  const cfg = normalizePrinterConfig(s.printerScreens ?? s.printerLayout);
  cfg.screens.idle.objects = cfg.screens.idle.objects.filter((o) => o.type !== "logo");
  await db.update(tenantSettings).set({ printerScreens: cfg }).where(eq(tenantSettings.organizationId, org.id));
  await enqueueConfigChangedForOrg(org.id, "hil-cleanup").catch(() => {});
  console.log("removed injected logo widget from Roastwell idle screen");
  process.exit(0);
}
main();
```

- [ ] **Step 2: Run it, then delete the script**

Run:
```bash
npx tsx lib/db/_cleanup_hil_logo.ts
rm -f lib/db/_cleanup_hil_logo.ts
```
Expected: prints `removed injected logo widget from Roastwell idle screen`. (Do NOT commit the throwaway script.)

- [ ] **Step 3: (no commit — data-only)**

This task changes prod data only; there is nothing to commit.

---

## Self-Review notes

- **Spec coverage:** relabel (Task 1); `createBrandNameObject` (Task 1); singleton-guarded add button (Task 2); deletable (Task 2); no cap impact / no `sanitizeScreen` change (verified — `logo` already a widget singleton); preview/firmware/payload unchanged (no task needed); test injection cleanup (Task 3).
- **Type consistency:** `createBrandNameObject(z)`, `hasBrandName`, `addBrandName` used identically across tasks.
- **No new HIL / firmware:** confirmed — `logo` rendering was already verified on hardware 2026-06-25.
