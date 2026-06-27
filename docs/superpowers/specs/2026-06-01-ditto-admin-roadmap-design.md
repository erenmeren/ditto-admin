# Ditto Admin — Product Roadmap

_Last updated: 2026-06-27_

> **Terminology note.** This product has been renamed twice since the original
> roadmap: **kiosk → printer** (2026-06-13) and **receipt → document**
> (2026-06-27, cloud + firmware). This doc uses the current terms (printer,
> document). Some DB columns deliberately keep the old names
> (`tenantSettings.perPrintPriceCents`, the `kiosk`-era columns) to avoid
> churn — that's intentional and not a gap.

## Shipped since the original roadmap (2026-06-21 → 2026-06-27)

Major deliveries that landed after this roadmap was first written, in addition
to the per-phase ✅ markers below:

- **Device Settings (cloud + firmware M7), 2026-06-21** — org-wide QR-visible
  duration, screen brightness, sleep/wake + inactivity timeout, and an on-device
  Settings PIN. Set under Branding (`/tenant/device-settings`), stored in
  `tenant_settings`, delivered via `/api/device/config` (ETag-versioned) +
  `config-changed` broadcast. Honored on-device (brightness→backlight, sleep
  with wake-on-touch/document, SHA-256-PIN-gated Settings menu). HW-verified.
- **Prepaid credits + public device-trigger API, 2026-06-26** — append-only
  `credit_ledger` (reserve→settle→release) + `credit_balance` cache, scoped API
  keys, a public `POST /api/v1/devices/{id}/trigger`, Stripe self-serve credit-pack
  top-up, and the admin/tenant credit-usage surfaces. Firmware Spec B (device
  receives the trigger, shows the QR, ACKs to settle the hold) HW-verified.
- **Resend transactional email, 2026-06-27** — live API key wired; alert + signup
  emails send. Domain verification still pending (sends to non-`erenaltan`
  recipients are gated on it).
- **Receipt → Document rename, 2026-06-27** — total rename across DB, public API
  (`/api/v1/documents`), the QR route (`/d/{token}`), webhooks
  (`document.created`/`document.downloaded`), UI, and the firmware. No back-compat.

## Context

Ditto is a multi-tenant, digital-document SaaS. Printers replace paper documents
with a QR code customers scan to download a private digital document. The admin
console is **feature-complete** today: Better Auth (org = tenant) with self-serve
signup, FK-scoped multi-tenancy, device provisioning + pairing → ingest →
document → public-token delivery, R2 image storage, monthly billing generation,
reports/CSV export, and both tenant + platform-admin dashboards.

This roadmap covers the arc from **launch-readiness → billing → feature
expansion → long-term vision**. Phases past 0 set direction and each get their
own spec → plan cycle when picked up.

**Guiding principle:** a secure, stable, production-ready system comes *before*
payment collection and billing automation. Phase ordering reflects that.

---

## 🚀 Phase 0 — Launch Readiness

The delta between "feature-complete" and "safe for real customers." **Largely
shipped** — remaining gaps are the external-account/ops items called out below.

### 1. Real landing page — ✅ SHIPPED
`app/page.tsx` is a real entry page ("Paper documents, gone.") that routes to
`/login` / `/signup`; authenticated visitors redirect to their dashboard by role.

### 2. Harden `/api/ingest` — ✅ SHIPPED
Per-device-key rate limiting (`lib/rate-limit.ts`, atomic UPSERT), payload size
caps on the uploaded document image, and early rejection of unknown/paused
devices are all in place. The shared API guard lives in `lib/api/guard.ts`.

### 3. Email verification — ⏳ MOSTLY (gated on domain)
Resend is wired and sending (Phase-0 activation, 2026-06-27). Flipping
`requireEmailVerification` to `true` for self-serve signups is gated on Resend
**domain verification** (today only `erenaltan@…` receives). Backfill for seeded
prototype users still to confirm.

### 4. Observability — ⏳ CODE COMPLETE (awaiting Sentry account)
Sentry is wired manual/init-only (`lib/observability.ts`), env-gated — absent
`SENTRY_DSN` the SDK no-ops. `beforeSend` scrubs secrets (device keys, `/d/`
tokens). Structured logging is on the critical paths. **Open:** create the Sentry
project + set the DSN to activate; add uptime/health alerting.

### 5. Backup & restore strategy — ⏳ OPEN
Confirm Neon PITR is enabled + document RPO/RTO; verify R2 lifecycle/versioning so
document images aren't silently lost; write a *tested* restore runbook.

### 6. Smoke test suite — ✅ SHIPPED (and grown)
The repo went from zero tests to a 254-test pure-function suite (73 files) covering
webhooks, billing, credits, search, serialization, and the env/observability glue.
(End-to-end ingest→token→download flow tests remain a nice-to-add.)

### 7. Deploy config — ✅ SHIPPED
Production env validation (`lib/env.ts`), Vercel deploy live
(`ditto-admin-brown.vercel.app`), and a health-check endpoint (`/api/health` +
`/api/cron/health`). `serverExternalPackages` survives the pipeline.

---

## 💳 Phase 1 — Close the Billing Loop

Two parallel money paths now exist: **prepaid credits** (shipped — the device-
trigger model) and **monthly invoices** (generation only — collection still open).

- **Stripe integration** — ✅ SHIPPED for credits (self-serve credit-pack
  purchase + webhook reconciliation, **test mode**). Live keys deferred to
  project completion (user decision). Invoice *collection* via Stripe is still
  open.
- **Invoice lifecycle** — ⏳ PARTIAL. The `invoice.status` enum already carries
  `draft`/`sent`/`paid`/`overdue`/`void`. **Open:** the transitions that drive
  them (auto-`overdue`, payment receipts) and a pay action.
- **Subscription / usage enforcement** — ⏳ OPEN. Credits already gate device
  triggers (reserve→settle), but grace periods, account suspension, and ingest
  throttling for unpaid *invoice* accounts are not built.
- **Tenant-facing billing** — ⏳ PARTIAL. Tenants see their credit usage; paying
  their own invoices (billing is platform-admin-only today) is still open.
- **Audit logs (start)** — ✅ STARTED. Best-effort audit logging captures
  significant actions (incl. device commands enqueued) with actor/action/target/
  metadata.

---

## 📈 Phase 2 — Feature Expansion

- **Audit logs (full)** — ⏳ PARTIAL (foundation shipped in Phase 1; the
  tenant/platform-facing activity trail UI is open).
- **Tenant** — ✅ document search/filtering (`lib/documents-search.ts`, keyset
  pagination) and ✅ live branding preview (the printer-preview surface) are
  shipped. ⏳ Open: team-member invites (org plugin already supports membership),
  per-store analytics.
- **Platform admin** — ⏳ tenant health dashboard, usage-based alerts
  (`lib/alerts-sync.ts` exists as a start), fleet-wide status views.
- **Device** — ✅ **Device Settings shipped 2026-06-21** (see banner). Device
  commands + ACK plumbing exist (trigger/show_qr), and `firmwareVersion` is
  reported on ingest. ⏳ Open: remote pause/reboot UI, offline detection,
  first-class firmware/version tracking in the console.

---

## 🔭 Phase 3 — Long-Term Vision

- **Public API + webhooks** — ✅ SHIPPED EARLY. `app/api/v1/{documents,devices,
  usage}` + `openapi.json`, and signed webhook delivery with retry
  (`lib/webhooks/`), so tenants can pull their own document data and subscribe to
  `document.*` events.
- Customer-facing document features (loyalty, opt-in marketing, return/warranty
  lookup off the document token) — ⏳ OPEN.
- Multi-region R2, white-label custom domains per tenant — ⏳ OPEN.

---

## Out of scope (for now)

- Mobile apps (web-first; documents are already mobile-web by nature)
- Non-Stripe payment processors

(The printer firmware lives in its own repo, **ditto-firmware**, with its own
milestone roadmap — it is no longer "out of scope" for the product but is tracked
separately.)

## Sequencing notes

- **Don't move invoice collection ahead of the Phase 0 ops gaps** (Sentry DSN,
  backup/restore runbook, email-domain verification). Hardening and recoverability
  gate the remaining billing work.
- Each phase past 0 gets its own `spec → plan → implementation` cycle when started.
- **Standing ops items (user-owned):** Stripe dashboard meter `event_name` →
  `documents`; Resend domain verification; Stripe test → live keys (deferred to
  project completion).
