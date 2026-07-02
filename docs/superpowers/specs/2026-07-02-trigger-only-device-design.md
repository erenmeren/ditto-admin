# Trigger-only Device — Strip the Document / Print / R2-Ingest Stack

**Date:** 2026-07-02
**Status:** Draft — awaiting user review
**Author:** Eren (via brainstorming session)

## Problem / Motivation

Ditto today has **two** ways a device produces a customer-facing QR:

1. **Legacy print/ingest flow** — the device renders a document (ESC/POS-style image),
   `POST /api/ingest` uploads it to R2, a `document` row + token is created, and the
   public `/r/{token}` or `/d/{token}` page serves the image. Phases 3A–3C
   (branded page, return/warranty window, email-me-this-document, magic-link
   recovery, marketing contacts) are all built on top of this.
2. **Trigger flow** — `POST /api/v1/devices/{deviceId}/trigger` enqueues a
   `show_qr` command with a caller-supplied URL; the device renders a QR from it,
   ACKs, and 1 prepaid credit settles.

We are dropping flow (1) entirely. **The cloud's only job becomes: receive a
trigger request and have the device display a QR from the caller's URL.**
Customers who want a document host it themselves and pass us its URL. No
document is created, stored, or served by Ditto.

## Goals

- Keep exactly one device-activation path: the trigger API.
- Adopt the caller-friendly request shape `{ deviceId, payloadType, value }`.
- Remove the document subsystem and everything Phase 3A–3C built on it.
- Do **not** break the already-shipped Spec-B firmware (it parses the internal
  `show_qr` command payload).
- Keep prepaid **credits + Stripe top-ups** as the live payment path.

## Non-Goals

- **Billing rework.** The per-print invoice/dunning machinery (`billing-engine.ts`,
  `invoice` table, `billing/billing-cron.ts`, Phase 1C transition emails) is left
  **in place, untouched**, per explicit decision. With no documents created it
  meters $0 until a later, dedicated billing pass. Consequently the `document`
  table itself is **kept (vestigial)** so billing keeps compiling.
- **Firmware repo (`ditto-firmware`).** Its receipt-render/upload/print pipeline
  becomes dead code but is out of scope for this repo's spec; strip it there
  separately.

## Design

### 1. Trigger API — new public request shape

- **New endpoint:** `POST /api/v1/devices/trigger`
  Body: `{ "deviceId": string, "payloadType": "url", "value": string }`
  - `payloadType` — only `"url"` supported today; the field exists so future
    payload types can be added without another contract break.
  - `value` — an `http:`/`https:` URL, length 1–2048 (reuses the existing
    `isValidUrl` check from `lib/trigger-actions.ts`).
- **Internal mapping (firmware compatibility):** the handler maps
  `payloadType:"url" + value` → the **unchanged** internal device command
  `{ action: "show_qr", payload: { url: value } }`. The device polls
  `/api/device/commands` and receives the identical shape it already parses, so
  **no firmware change is required.**
- **Preserved behavior** (all already implemented on the current route, moved to
  the new one): API-key auth via `guardApiRequest`, `devices:trigger` scope check,
  required `Idempotency-Key` header + idempotency replay, credit
  reserve→settle→release, `device_offline`/`device_not_found`/`insufficient_credits`
  error paths, 60s command TTL, lazy expired-hold reconciliation.
- **Removed:** the old path-param route `app/api/v1/devices/[deviceId]/trigger/`.
- `lib/trigger-actions.ts` stays as the internal action/cost source of truth
  (`show_qr` = 1 credit). `validateTriggerBody` is refactored/renamed to validate
  the new external `{ deviceId, payloadType, value }` shape (deviceId presence +
  URL validity), still returning the internal action/payload.
- **OpenAPI:** `app/api/v1/openapi.json` updated to the new endpoint + schema;
  document endpoints removed from it.

### 2. Remove the document subsystem

**Routes / endpoints removed:**
- `app/api/ingest/`
- `app/api/v1/documents/` and `app/api/v1/documents/[id]/`
- `app/(public)/r/[token]/`
- `app/(public)/d/[token]/` and the entire `/d/lookup/*` recovery flow

**Lib removed:**
- `lib/documents.ts`, `lib/documents-search.ts`, `lib/lookup/*`
- document-specific view-model functions in `lib/data.ts`
  (`getDocument*`, `listDocumentsByCursor`, document counts/series helpers, etc.)
- the `putDocument` / `presignedDocumentUrl` aliases in `lib/storage.ts`
  (R2 core — `putObject`, `presignedGetUrl`, `deleteObject` — **stays**; it still
  serves logos, firmware binaries, and printer-config icons)
- `lib/api/serialize.ts` document serializer (if unused elsewhere)

**Phase 3A–3C features removed** (all sit on the document / `/d` page):
branded document page, return/warranty window, email-me-this-document,
magic-link recovery, marketing contacts.

**Schema / migration:**
- Drop tables `documentContact` and the `lookup` table(s) (Phase-3 additions),
  plus the Phase-3 columns added to `tenantSettings` (logo/brand-color/support/
  return-window/warranty settings that only fed the `/d` page — audit each; keep
  any also used by the device QR screen branding).
- **Keep** the `document` table definition (billing dependency).
- New Drizzle migration; strip generated SQL to just these drops
  (per the known snapshot-drift gotcha).

**Tenant / admin UI:**
- Remove `/tenant/documents` (list, `[documentId]` detail, search).
- Remove the tenant marketing-contacts page and document-recovery surfaces.
- **Metrics (DEFAULT — repoint to QR triggers):** replace every "documents
  delivered" count/series in the tenant dashboard, `/tenant/analytics`,
  `/tenant/reports`, store detail, and admin home with trigger counts. Source of
  truth = the **credit ledger** (settled `show_qr` charges) — it persists as a
  billing-grade record and supports historical time-series; `deviceCommand`
  (type = `trigger`, also persisted, no reaper) is the fallback for pending/failed
  breakdowns. Tenants see "QRs shown" instead of "documents." *(Chosen by default
  while user was away; revertible to "just remove the metric cards" on review.)*

### 3. Billing — unchanged (explicit decision)

`billing-engine.ts`, `invoice` table, `billing/billing-cron.ts`, dunning, and
Phase 1C transition emails are **left as-is**. They read the (now-empty)
`document` table and meter $0 until the separate billing pass. `perPrintPriceCents`
stays in the schema for the same reason. Prepaid **credits + Stripe credit-pack
top-ups** remain the live payment path (each trigger charges 1 credit).

### 4. Kept unchanged

Device claim / commands / ACK / config / firmware / OTA; prepaid credit ledger,
holds, and credit-holds cron; R2 core (non-document assets); branding/logo;
**printer-display config** (`printer-layout` / `printer-geometry` / `printer-icons`
+ `device-config`) — the device still renders a *branded QR screen*, which is
display config, not document printing *(DEFAULT — keep; revertible on review)*;
stores; audit log; auth / orgs; health + usage crons.

## Data Flow (after)

```
External system                Ditto cloud                     Device
---------------                -----------                     ------
POST /api/v1/devices/trigger
  { deviceId, payloadType:"url",
    value: "https://host/doc" }
        ──────────────────────▶  auth + scope + idempotency
                                 reserve 1 credit
                                 insert deviceCommand
                                   { action:"show_qr",
                                     payload:{ url:value } }
                                 202 { id, status:"queued" }
                                                        ◀── GET /api/device/commands (poll)
                                                            render QR from url
                                                        ── POST .../ack ──▶ settle credit
```

## Error Handling

Unchanged from the current trigger route — `invalid_request` (bad body/URL),
`missing_idempotency_key`, `insufficient_scope`, `device_not_found`,
`device_offline`, `insufficient_credits`, `conflict`, `internal_error`.

## Testing

- **Add / update:** trigger-body validation tests for the new
  `{ deviceId, payloadType, value }` shape; an endpoint test for
  `POST /api/v1/devices/trigger` covering the full reserve→enqueue→idempotency
  path and internal `show_qr` mapping.
- **Remove:** ingest, document, lookup, Phase-3, and marketing-contacts test
  suites alongside their code. Overall test count will drop from 332.
- **Keep:** credit ledger/holds, device-command/config/status, API-scope, and
  billing tests (billing untouched).

## Risks / Open Questions

- **Vestigial `document` table + billing metering $0** is a deliberate interim
  state; flagged so the later billing pass is not forgotten.
- Confirm which `tenantSettings` columns are safe to drop (only `/d`-page-facing
  ones) vs. those reused by the device QR-screen branding.
- The two "away" defaults (repoint metrics to triggers; keep printer-display
  config) are marked inline and should be confirmed at review.
