# Ditto Admin — Product Roadmap

_Last updated: 2026-06-21_

> **Shipped 2026-06-21 — Device Settings (cloud + firmware).** Org-wide device
> policies — QR-code visible duration, screen brightness, screen sleep/wake +
> inactivity timeout, and an on-device Settings PIN — set in the tenant console
> under Branding (`/tenant/device-settings`), stored in `tenant_settings`,
> delivered to every device via `/api/device/config` (ETag-versioned) and a
> `config-changed` broadcast. Deployed to production. The device now honors them
> (ditto-firmware **M7**: brightness→backlight, screen sleep with wake-on-touch/
> receipt, long-press SHA-256-PIN-gated Settings menu) — merged, flashed, and
> HW-verified. See Phase 2 → Device below.

## Context

Ditto is a multi-tenant, digital-receipt SaaS. Kiosks replace paper receipts
with a QR code customers scan to download a private digital receipt. The admin
console is **feature-complete** today: Better Auth (org = tenant) with self-serve
signup, FK-scoped multi-tenancy, device provisioning + pairing → ingest →
receipt → public-token delivery, R2 image storage, monthly billing generation,
reports/CSV export, and both tenant + platform-admin dashboards.

This roadmap covers the arc from **launch-readiness → billing → feature
expansion → long-term vision**. Phase 0 is detailed for immediate execution
(next 1–2 sprints); later phases set direction and will each get their own
spec → plan cycle when picked up.

**Guiding principle:** a secure, stable, production-ready system comes *before*
payment collection and billing automation. Phase ordering reflects that.

---

## 🚀 Phase 0 — Launch Readiness _(Sprint 1–2, detailed)_

The delta between "feature-complete" and "safe for real customers."

### 1. Real landing page
Replace the Next.js boilerplate in `app/page.tsx` (currently "edit page.tsx" +
Next logo) with a real entry page that routes to `/login` and `/signup` and
states what Ditto is. Authenticated visitors redirect to their dashboard by role.

### 2. Harden `/api/ingest`
The ingest endpoint is the one public surface authenticated only by a device
bearer key (not a user session). Before real devices hit it:
- Per-device-key rate limiting (token bucket / sliding window)
- Payload size caps on the uploaded receipt image (reject oversized/invalid)
- Reject unknown/paused devices early (already partially handled — verify)

### 3. Email verification
Flip `requireEmailVerification` to `true` for self-serve signups and wire a
transactional email sender (Resend recommended). Seeded prototype users may
need a one-time backfill/verified flag. Covers signup confirmation + the
hooks for password reset.

### 4. Observability
- Error tracking + alerts (Sentry) across server actions, the ingest route, and
  the public receipt route
- Structured logging on the critical paths (ingest, claim, receipt download)
- Basic uptime/health alerting

### 5. Backup & restore strategy
- Confirm Neon point-in-time-restore is enabled and document the recovery
  procedure (RPO/RTO targets)
- Verify R2 object lifecycle/versioning so receipt images aren't silently lost
- Document a tested restore runbook (not just "backups exist")

### 6. Smoke test suite
First tests in the repo (none exist today). Cover the critical paths:
- ingest → receipt row created → public token view → `ready → downloaded` flip
- auth/role gates: `requireTenant` / `requirePlatformAdmin` reject the wrong role
- device claim consumes the one-time pairing code and returns the raw key once

### 7. Deploy config
Production env validation (`lib/env.ts`), Vercel deploy, health-check endpoint,
and verify `serverExternalPackages` / webpack build flag survive the deploy
pipeline (this has silently reverted before).

---

## 💳 Phase 1 — Close the Billing Loop _(this quarter)_

Invoices currently generate (`draft` / `sent` / `paid`) but nothing collects.

- **Stripe integration** — turn `sent` invoices into collectible payments;
  payment webhooks reconcile invoice status
- **Invoice lifecycle** — add `overdue` and `void` states; payment receipts
- **Subscription enforcement** — grace periods, account suspension, and usage
  limits (e.g. ingest throttling/blocking) for unpaid accounts
- **Tenant-facing billing** — tenants can see and pay their own invoices
  (billing is platform-admin-only today)
- **Audit logs (start)** — begin capturing billing- and account-state changes;
  the foundation for the broader audit log in Phase 2

---

## 📈 Phase 2 — Feature Expansion

- **Audit logs (full)** — tenant- and platform-level activity trail across
  devices, members, billing, and settings. Pulled early here for enterprise
  readiness.
- **Tenant** — receipt search/filtering, live branding preview, team member
  invites (the org plugin already supports membership), per-store analytics
- **Platform admin** — tenant health dashboard, usage-based alerts, fleet-wide
  status views
- **Device** — remote pause/reboot, firmware/version tracking, offline detection.
  ✅ **Device Settings shipped 2026-06-21** — org-wide brightness, screen
  sleep/wake, QR-visible duration, and an on-device Settings PIN, configured in
  the tenant console and honored on-device (firmware M7).

---

## 🔭 Phase 3 — Long-Term Vision

- Customer-facing receipt features (loyalty, opt-in marketing, return/warranty
  lookup off the receipt token)
- Multi-region R2, white-label custom domains per tenant
- Public API + webhooks so tenants can pull their own receipt data

---

## Out of scope (for now)

- Mobile apps (web-first; receipts are already mobile-web by nature)
- The kiosk firmware itself (this repo is the admin console + ingest API)
- Non-Stripe payment processors

## Sequencing notes

- **Don't move payments ahead of Phase 0.** Hardening, observability, and
  recoverability gate the billing work.
- Each phase past 0 gets its own `spec → plan → implementation` cycle when
  started; Phase 0 items are small enough to plan as one batch.
