# Top-Risks Remediation Plan (2026-06-12)

Fix the five highest-risk areas surfaced in the codebase review. Branch: `fix/top-risks`.
Subagent-driven, sequential where files overlap, parallel where disjoint.

Decisions (from the user):
- Rate limiter → **Neon Postgres counter table** (atomic, cross-instance).
- Signup atomicity → **Both**: compensating cleanup for the Better-Auth steps + a real
  transaction (Neon websocket Pool) around our own writes.

DB note: subagents **generate** migrations (`drizzle-kit generate`, offline) but DO NOT
apply them. Applying (`npm run db:migrate`) is left to the user (hard-to-reverse, hits live Neon).

---

## WS1 — Durable usage metering (revenue integrity) — RISK #1
**Problem:** `app/api/ingest/route.ts` reports Stripe usage fire-and-forget (`.catch()`),
never retries, no reconciliation → silent revenue loss.

- Add `usage_event` table to `lib/db/schema.ts`:
  `id`, `organizationId` (FK), `receiptId` (FK, **unique** → idempotent), `stripeCustomerId` (nullable),
  `status` enum `pending|reported|skipped`, `attempts` int, `createdAt`, `reportedAt` (nullable),
  index on `(status, createdAt)`.
- New `lib/billing/usage-metering.ts`: `recordUsageEvent(...)` (insert pending in the same path as
  the receipt insert), `reportUsageEvent(...)` (attempt Stripe meter, mark reported/bump attempts).
  Keep a pure status-transition helper with colocated tests.
- `app/api/ingest/route.ts`: insert the `usage_event` row alongside the receipt; attempt an immediate
  report; on failure leave it `pending` (no throw). `skipped` when Stripe/customer is absent.
- New cron `app/api/cron/usage/route.ts` (CRON_SECRET-gated like the others): sweep `pending` rows
  past a small delay, report to Stripe, mark `reported`; cap attempts. Add to `vercel.json` crons.
- Tests: pure transition logic.
**Files:** schema.ts, new lib/billing/usage-metering.ts (+test), ingest route, new cron route, vercel.json, generated migration.

## WS2 — Rate limiter backed by Neon — RISK #2  (run AFTER WS1; shares schema.ts + ingest route)
**Problem:** in-memory `Map` limiter is per-instance → effectively unenforced on serverless.

- Add `rate_limit` table to `lib/db/schema.ts`: `key` (PK), `windowStart` (timestamp), `count` (int).
- Rewrite `lib/rate-limit.ts`: keep a pure window-math helper (testable); add async
  `checkRateLimit(key, {limit, windowMs})` doing a **fixed-window atomic UPSERT**
  (`insert ... on conflict do update set count = case when window matches then count+1 else 1 end`,
  returning the post-increment count + window) and returning `{allowed, retryAfterMs}`. Fail-open on DB error
  (log) so a DB blip can't hard-block ingest.
- Callers become `await`: `app/api/ingest/route.ts`, `lib/api/guard.ts`.
- Update `lib/rate-limit.test.ts` to test the pure window helper; keep `__resetRateLimit` semantics where used.
**Files:** schema.ts, rate-limit.ts (+test), ingest route, lib/api/guard.ts, generated migration.

## WS3 — Super-admin data path: aggregate in SQL — RISK #3  (disjoint files; parallel-safe)
**Problem:** `loadAllOrgs()` pulls every org's full device + receipt set into app memory for
`getAdminOverview`, `getBillingOverview`, `getAllDevices`, `getTenantSummaries` → memory/latency cliff.

- Replace the all-rows-in-memory aggregation with SQL `GROUP BY` queries (counts, sums, monthly/daily
  buckets) in `lib/data.ts`, reusing the existing pure `analytics` shaping helpers. Keep the same
  return types/shapes so pages and tests are unchanged. Per-tenant receipt counts, device status
  rollups, and revenue come from grouped queries, not per-row JS reduction.
- Preserve exact view-model output (cents→dollars, UTC buckets) — verify against current behavior.
**Files:** lib/data.ts (+ any colocated test).

## WS4 — Webhook SSRF: resolve DNS before allowing — RISK #4  (disjoint files; parallel-safe)
**Problem:** `lib/webhooks/url-guard.ts` only inspects string/IP literals → a hostname resolving to a
private/metadata IP (DNS rebinding) passes.

- Add async resolution: resolve the URL host (`dns.lookup`/`promises.resolve`) and reject if ANY
  resolved address is private/loopback/link-local/ULA (reuse the existing literal checks against each
  resolved IP). Keep the pure literal validator for the create-time form check; add an async
  `assertAllowedWebhookUrl` used in `lib/webhooks/deliver.ts` `attemptDelivery` before `fetch`.
- Document the residual TOCTOU (full rebinding protection needs IP pinning on the socket) in a comment.
- Tests for the literal validator stay; add tests for the resolved-IP classifier (pure part).
**Files:** lib/webhooks/url-guard.ts (+test), lib/webhooks/deliver.ts.

## WS5 — Signup atomicity — RISK #5  (disjoint files; parallel-safe)
**Problem:** `lib/actions/register.ts` writes user→org→membership→settings with no transaction → orphans.

- Add a transactional DB client using Neon websocket **Pool** (`drizzle-orm/neon-serverless`) — a new
  export (e.g. `lib/db.ts` `dbTx`/`withTransaction`) reusing the same schema. (`@neondatabase/serverless`
  is already a dependency.)
- Wrap our own writes (ensure-membership + seed tenant_settings) in a single transaction.
- Compensating cleanup: if a step fails after the Better-Auth user/org was created, delete the orphaned
  org (cascades membership/settings) and the just-created user, then return a clean error. Make the flow
  idempotent so a retry is safe.
**Files:** lib/db.ts (new tx client), lib/actions/register.ts.

---

## Integration (after all workstreams)
- `npx tsc --noEmit` clean.
- `npm test` (vitest) green; update any tests touched by signature changes.
- `npm run build` succeeds.
- Surface to user: migrations generated but NOT applied — they should run `npm run db:migrate`,
  and (WS1) add the `/api/cron/usage` schedule is in `vercel.json`.
