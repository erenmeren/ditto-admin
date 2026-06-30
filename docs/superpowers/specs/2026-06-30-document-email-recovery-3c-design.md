# Phase 3C — Document email & lost-link recovery

**Status:** Design approved 2026-06-30
**Phase:** 3C (third Phase 3 customer-facing piece, after 3A branded page + 3B return/warranty window)

## Problem

A customer scans a printer's QR, lands on the branded public document page
(`/d/[token]`), which now shows return/warranty windows (3A/3B). The token *is*
the only way back to that document — lose the link and the document is gone.
Documents carry **no customer-identifiable data** (the `document` row and
`DocumentMetadata` hold render facts only), so there is nothing to "look up" by
today.

3C lets a customer **attach their email to a document** so they can (a) get it
mailed to themselves and (b) later recover all the documents they emailed
themselves from that merchant — plus an optional marketing opt-in the merchant
can use.

## Honest scope boundary

Recovery can **only** surface documents a customer explicitly emailed themselves.
We cannot retroactively link anonymous past scans. "Find my documents" means
"find the documents I asked to be emailed." This is by design.

## Email-delivery dependency

The whole feature is email-dependent. `RESEND_API_KEY` is not configured, and
`lib/email.ts` `sendEmail` no-ops (logs) without it. We build 3C **fully but
inert**, identical to email-verification and alert digests: forms render,
associations/consent persist, sends gracefully no-op. The moment a Resend domain
is verified and the key is set, emails deliver with **zero code changes**.

## Decisions

| Area | Decision |
|---|---|
| Build scope | Both "email me this document" + magic-link recovery, gated/inert |
| Marketing opt-in | Captured **and** surfaced: `/tenant/contacts` page (list + CSV) |
| Recovery scope | **Per-merchant** (org-scoped) — never leak other merchants a customer used |
| Magic link | Single-use, 30-min TTL, stored as SHA-256 hash (raw only in the URL) |
| "Email me" payload | A **link** back to `/d/{token}` (token is capability), not an image attachment |
| Privacy boundary | Tenants see only *consented* (`marketing_contact`) rows, never transactional sends |

## Data model (one migration, three tables)

### `document_contact` — transactional email↔document link
One row per "email me this document" send. **Never shown to tenants.**
- `id` text PK
- `organizationId` text → `organization` (cascade)
- `documentId` text → `document` (cascade)
- `email` text (lowercased, normalized)
- `createdAt` timestamp
- Indexes: `(organizationId, email)` (recovery query), `(documentId)`

### `marketing_contact` — consent record (the tenant-visible list)
Written only when the customer ticks "Keep me posted".
- `id` text PK
- `organizationId` text → `organization` (cascade)
- `email` text (lowercased)
- `optInAt` timestamp
- `createdAt` timestamp
- **Unique `(organizationId, email)`** — upsert on re-opt-in (idempotent)

### `lookup_token` — magic-link grant
- `id` text PK
- `organizationId` text → `organization` (cascade)
- `email` text (lowercased)
- `tokenHash` text (SHA-256 of the raw token; reuse `lib/ids.ts` hashing pattern)
- `expiresAt` timestamp (createdAt + 30 min)
- `consumedAt` timestamp nullable (single-use)
- `createdAt` timestamp
- Index: `(tokenHash)`

## Flows & routes

### Public — new

**`requestDocumentEmail({ token, email, optIn })`** (server action)
- Rate-limited per-IP (`lib/rate-limit-window.ts`).
- Look up document by token; if unknown, still return generic success (no oracle).
- Insert `document_contact`.
- If `optIn`: upsert `marketing_contact` (`onConflictDoUpdate` stamps `optInAt`).
- `sendEmail` with a branded-light template linking to `{BETTER_AUTH_URL}/d/{token}`.
- Always returns generic `{ ok: true }`.

**`/d/lookup/[orgId]`** (public page) — email entry form.
**`requestLookupLink({ orgId, email })`** (server action)
- Rate-limited per-IP.
- Create `lookup_token` (raw token via nanoid; store hash).
- `sendEmail` magic link `{BETTER_AUTH_URL}/d/lookup/{orgId}/{rawToken}`.
- **Always** returns "If we have documents for that email, we've sent a link"
  (no enumeration), whether or not any `document_contact` rows exist.

**`/d/lookup/[orgId]/[token]`** (public page) — recovery listing.
- Hash the URL token, find an unconsumed, unexpired `lookup_token` for `orgId`.
- Invalid/expired/consumed → friendly "link expired, request a new one" screen.
- Valid → stamp `consumedAt`, list `document_contact` rows for `(orgId, email)`,
  each showing `createdAt` + `coverageStatus()` windows + a link to `/d/{token}`.

### Tenant — new

**`/tenant/contacts`** (owner/admin) — reads `marketing_contact` for the active
org; sortable table + CSV export. New read fns in `lib/data.ts`. Role-gated like
other `/tenant/*` pages.

### Modified

`/d/[token]` page gains a client component section: email input + "Keep me
posted" checkbox (default **unchecked**) + a "Find my other documents" link to
`/d/lookup/{orgId}`.

## Pure / testable modules

- `lib/lookup/token.ts` — generate raw token, hash, verify (expiry + consumed)
  given an injected `now` (IO-free, unit-tested like `lib/branding/coverage.ts`).
- `lib/lookup/email-templates.ts` — HTML builders for the two emails
  (HTML-escaped; pattern from `lib/billing/invoice-emails.ts`).
- `lib/lookup/normalize.ts` — email normalization + generic-response helper.

Reused as-is: `sendEmail`, `lib/rate-limit-window.ts`, `coverageStatus`,
`getDocumentByToken`, `lib/ids.ts` hashing, `organization` (uses opaque
`organizationId` in recovery URLs — `slug` is nullable so not relied on).

## Security / abuse

- Both public POST paths rate-limited per-IP.
- No email enumeration: generic responses on both send paths.
- Magic links single-use, 30-min TTL, hashed at rest.
- Marketing checkbox defaults unchecked (explicit consent).
- No new presigned-URL surface; recovery links to `/d/{token}` which keeps the
  fresh-presign-on-view model.
- New customer-email PII is org-scoped and cascade-deleted with the org/document.

## Testing

- **Unit:** token TTL/single-use, email normalization, template escaping,
  generic-response helper.
- **Integration (throwaway tsx vs live Neon — per the timestamp-TZ lesson):**
  `document_contact` insert + recovery query, `marketing_contact` upsert
  idempotency, lookup-token consume-once.
- **Manual QA (Playwright, once Resend set):** full email-me → magic-link →
  recovery loop; tenant contacts list + CSV.

## Out of scope / deferred

- Image-attachment emails (link only).
- Cross-merchant recovery.
- Loyalty points / programs (separate future phase).
- Exposing contacts via `/api/v1` (the page + CSV cover this cut).
