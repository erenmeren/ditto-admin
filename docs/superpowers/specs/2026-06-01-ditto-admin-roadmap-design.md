# Ditto Admin — Product Roadmap

_Last updated: 2026-07-01_

> **Terminology note.** This product has been renamed twice since the original
> roadmap: **kiosk → printer** (2026-06-13) and **receipt → document**
> (2026-06-27, cloud + firmware). This doc uses the current terms (printer,
> document). Some DB columns deliberately keep the old names
> (`tenantSettings.perPrintPriceCents`, the `kiosk`-era columns) to avoid
> churn — that's intentional and not a gap.

## Shipped since the original roadmap (2026-06-21 → 2026-07-01)

**Phases 1, 2, and 3A–3C all completed between 2026-06-27 and 2026-07-01.** In
brief (details in each phase section below):

- **Phase 1 — Billing loop CLOSED** (1A payable Stripe invoices / hybrid
  collection, 1B dunning cron + `isOrgPaymentBlocked` enforcement, 1C tenant
  transition emails). Stripe still in test mode (live keys deferred by user).
- **Phase 2 — Feature expansion CLOSED** (2A device fleet ops + offline reconcile,
  2B per-tenant health rollup + offline email, 2C audit-log activity UI).
- **Phase 3A–3C shipped** — branded public document page (3A), return/warranty
  window (3B), document-email + single-use magic-link recovery + opt-in marketing
  contacts (3C, merged+deployed 2026-07-01). This closes the doc's old Phase 3
  "return/warranty lookup" and "opt-in marketing" open items.

Earlier deliveries (2026-06-21 → 2026-06-27), in addition to the per-phase ✅
markers below:

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
The repo went from zero tests to a **332-test** pure-function suite (49 files) covering
webhooks, billing, credits, search, serialization, coverage/return-window math,
lookup-token/normalize/email-template helpers, and the env/observability glue.
(End-to-end ingest→token→download flow tests remain a nice-to-add.)

### 7. Deploy config — ✅ SHIPPED
Production env validation (`lib/env.ts`), Vercel deploy live
(`ditto-admin-brown.vercel.app`), and a health-check endpoint (`/api/health` +
`/api/cron/health`). `serverExternalPackages` survives the pipeline.

---

## 💳 Phase 1 — Close the Billing Loop — ✅ COMPLETE

Both money paths now close end-to-end: **prepaid credits** (the device-trigger
model) and **monthly invoices** (generation → collection → dunning → receipts).

- **Stripe integration** — ✅ SHIPPED for credits AND invoices. Invoice
  *collection* now goes through Stripe as payable invoices (1A, hybrid
  collection, Net 14 pay-link). Still **test mode** — live keys deferred to
  project completion (user decision).
- **Invoice lifecycle** — ✅ SHIPPED. The billing cron (1B) generates invoices,
  auto-sends to card tenants, and sweeps overdue; `invoice.dueDate` drives
  auto-`overdue`; tenant transition emails (1C) fire on sent/failed/paid/overdue.
- **Subscription / usage enforcement** — ✅ SHIPPED. `isOrgPaymentBlocked` gates
  past-due (402) and suspended (403) accounts (fail-open), on top of the existing
  credit reserve→settle gating for device triggers.
- **Tenant-facing billing** — ✅ SHIPPED. Tenants see credit usage and can pay
  their own invoices via the hosted Stripe pay-link.
- **Audit logs (start)** — ✅ SHIPPED. Best-effort audit logging captures
  significant actions (incl. device commands enqueued) with actor/action/target/
  metadata. (Full activity-trail UI landed in Phase 2C.)

---

## 📈 Phase 2 — Feature Expansion — ✅ COMPLETE

- **Audit logs (full)** — ✅ SHIPPED (2C). Friendly action labels + humanizer,
  `getOrgAuditPage` pagination, tenant Activity table, and admin labels.
- **Tenant** — ✅ document search/filtering (`lib/documents-search.ts`, keyset
  pagination), ✅ live branding preview (the printer-preview surface),
  ✅ team-member invites, and ✅ per-store analytics are shipped.
- **Platform admin** — ✅ tenant health dashboard + per-tenant health rollup (2B),
  ✅ usage-based alerts (`lib/alerts-sync.ts`, daily health cron), and
  ✅ fleet-wide status views / device detail (2A) are shipped.
- **Device** — ✅ **Device Settings shipped 2026-06-21** (see banner). ✅ Device
  fleet ops + offline reconcile (2A, folded into the daily health cron),
  ✅ first-class firmware/version tracking (fleet firmware column/badge), and
  device commands + ACK plumbing (trigger/show_qr). ⏳ Open (minor): a dedicated
  remote pause/reboot UI.

---

## 🔭 Phase 3 — Long-Term Vision — 🟡 IN PROGRESS (3A–3C shipped)

- **Public API + webhooks** — ✅ SHIPPED EARLY. `app/api/v1/{documents,devices,
  usage}` + `openapi.json`, and signed webhook delivery with retry
  (`lib/webhooks/`), so tenants can pull their own document data and subscribe to
  `document.*` events.
- **Customer-facing document features** — 🟡 mostly shipped off the document token:
  - ✅ **3A branded document page** — tenant logo + brand-color accent +
    provenance + optional support block (merged 2026-06-28).
  - ✅ **3B return/warranty lookup** — public `/d/{token}` shows computed
    return-deadline + warranty-expiry from tenant-set windows (shipped to prod
    2026-06-29/30).
  - ✅ **3C document email + recovery + opt-in marketing** — email-me-this-document,
    single-use magic-link recovery (`/d/lookup/...`, interstitial-POST so
    prefetchers can't burn the link), and tenant marketing-contacts page + CSV
    export (merged+deployed 2026-07-01). Ships inert until Resend domain
    verification. ⏳ Open: **loyalty**.
- **Infra** — ⏳ OPEN: multi-region R2, white-label custom domains per tenant.

---

## Out of scope (for now)

- Mobile apps (web-first; documents are already mobile-web by nature)
- Non-Stripe payment processors

(The printer firmware lives in its own repo, **ditto-firmware**, with its own
milestone roadmap — it is no longer "out of scope" for the product but is tracked
separately.)

## Sequencing notes

- **Phases 1 and 2 are closed; Phase 3 is underway (3A–3C shipped).** The billing
  loop shipped in test mode, so the original "don't collect before hardening" gate
  is moot — what remains are the Phase 0 ops gaps themselves.
- **Remaining Phase 0 hardening (the real "next"):**
  - **Backup & restore runbook** (§5) — the only remaining item fully actionable
    without a new external account; do this next.
  - **Sentry DSN** (§4) — code-complete, needs the account + DSN.
  - **Email-domain verification** (§3) — flips `requireEmailVerification` on and
    lets Phase 3C customer emails actually deliver (today: `erenaltan@…` only).
- Remaining Phase 3 initiatives (loyalty; multi-region R2; white-label custom
  domains) each get their own `spec → plan → implementation` cycle when started.
- **Standing ops items (user-owned):** Stripe dashboard meter `event_name` →
  `documents`; Resend domain verification; Stripe test → live keys (deferred to
  project completion).
