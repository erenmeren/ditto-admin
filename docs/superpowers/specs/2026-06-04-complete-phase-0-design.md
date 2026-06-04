# Complete Phase 0 — Observability + Email Verification

_Date: 2026-06-04_

## Context

Phase 0 (Launch Readiness) is shipped except for two leftovers called out in
the roadmap (`docs/superpowers/specs/2026-06-01-ditto-admin-roadmap-design.md`):

1. **Observability** — no error tracking exists (no Sentry package, config, or
   `lib/env.ts` entry).
2. **Email verification** — `requireEmailVerification: true` is set in
   `lib/auth.ts` and `sendVerificationEmail` is wired, but both signup paths
   (`registerCompany` in `lib/actions/register.ts`, `acceptInviteSignup` in
   `lib/actions/members.ts`) deliberately set `emailVerified: true` and call
   `auth.api.signInEmail`, so the gate enforces nothing. Real "check your email"
   UX was deferred pending a Resend key.

This spec closes both, following the codebase's established **"wired but inert
until the env key is present"** convention (already used by Stripe billing and
`lib/email.ts`'s Resend wrapper).

### Decisions (locked during brainstorming)

- **Activation: wire for keys.** Build both fully; they activate automatically
  when `SENTRY_DSN` / `RESEND_API_KEY` are set, and preserve current dev/seed
  behavior when absent. No external accounts required to ship the code.
- **Email scope: verification only.** No password-reset flow (not a Phase 0
  line item; deferred).
- **Sentry depth: approach B (manual capture, init-only).** Use `@sentry/nextjs`
  but **skip `withSentryConfig`** so the fragile build pipeline (`next build
  --webpack`, `serverExternalPackages`) is untouched. Source-map upload and auto
  performance tracing are deferred follow-ups.

## Guiding principles

- **Zero overhead when inert.** With no DSN, `Sentry.init` is skipped so the SDK
  no-ops. With no Resend key, signup behaves exactly as it does today.
- **No build-pipeline changes.** `next.config.ts`, the `--webpack` flag, and
  `serverExternalPackages` are not modified. Any change there must be justified
  by a failing build smoke test, not assumed.
- **Never log secrets.** Captured Sentry context must redact the device bearer
  key and the receipt `token` (both are capabilities).

---

## Part 1 — Sentry observability (approach B)

### New files

- **`instrumentation.ts`** (repo root). Next 16 calls `register()` once per server
  instance. It dynamically imports the runtime-appropriate config:

  ```ts
  export async function register() {
    if (process.env.NEXT_RUNTIME === "nodejs") await import("./sentry.server.config");
    if (process.env.NEXT_RUNTIME === "edge") await import("./sentry.edge.config");
  }
  export const onRequestError = Sentry.captureRequestError;
  ```

  `Sentry.captureRequestError` is the SDK's documented manual `onRequestError`
  handler — it forwards every server/render/action error Next surfaces.

- **`instrumentation-client.ts`** (repo root). Browser-side `Sentry.init`, guarded
  on `NEXT_PUBLIC_SENTRY_DSN`. Captures unhandled errors in the dashboard React app.

- **`sentry.server.config.ts`** / **`sentry.edge.config.ts`**. Each calls
  `Sentry.init({ dsn, environment, tracesSampleRate })` **only if** the DSN is set;
  otherwise returns without initializing.

### Env (`lib/env.ts` + `.env.example`)

Add three optional vars (all `z.string().optional()`):

| Var | Purpose |
|---|---|
| `SENTRY_DSN` | Server/edge DSN. Absent → server SDK inert. |
| `NEXT_PUBLIC_SENTRY_DSN` | Browser DSN (must be `NEXT_PUBLIC_` to reach the client). Absent → client SDK inert. |
| `SENTRY_ENVIRONMENT` | Tag for events; defaults to `"development"`. |

### Critical-path capture

`onRequestError` covers framework-surfaced errors automatically. Add explicit
`Sentry.captureException(err, { tags, extra })` in the **catch** blocks of the
paths the roadmap names, with redacted context:

- `app/api/ingest/route.ts` — ingest failures (tag: `path: "ingest"`; extra:
  `deviceId`, `orgId` — **never** the bearer key).
- `claimDevice` in `lib/receipts.ts` — claim failures (tag: `path: "claim"`).
- `app/(public)/r/[token]/page.tsx` — receipt render/presign failures (tag:
  `path: "receipt"`; **never** the raw token — use `receiptId` if available).
- Server actions — a small wrapper/helper so action catch blocks report to Sentry
  without each duplicating boilerplate.

### What we are NOT changing

`next.config.ts`, the webpack build flag, `serverExternalPackages`, and the
package's build script. No `withSentryConfig`. No source-map upload (stack traces
will be minified in prod — accepted; graduating to source maps is a follow-up).

---

## Part 2 — Email verification (real UX, inert until Resend key)

### Gate helper

A single source of truth: `emailVerificationEnabled()` → `Boolean(env.RESEND_API_KEY)`.
(Co-locate with `lib/email.ts` or `lib/auth.ts`.)

### Conditional signup behavior

In both `registerCompany` and `acceptInviteSignup`, branch on the gate:

- **Key present (verification live):** do **not** set `emailVerified: true`, do
  **not** auto-`signInEmail`. The org/membership/tenant-settings rows are still
  created (the user owns their org regardless of verification state). Return a
  result carrying `pendingVerification: true` and the email. Better Auth's
  `sendOnSignUp: true` already dispatched the verification email.
- **Key absent (current behavior, unchanged):** set `emailVerified: true` +
  `signInEmail`, return `{ ok: true }`. Dev, seed, and current production flows
  are byte-for-byte preserved.

The action result type gains an optional `pendingVerification?: boolean` and
`email?: string`.

### "Check your email" screen

- New route **`app/(auth)/verify-email/page.tsx`** (matching the existing auth
  route group). States: "We sent a verification link to **{email}**." with a
  **Resend** button calling `authClient.sendVerificationEmail({ email })`.
- The signup forms (company signup + invite signup) route to this screen instead
  of `/tenant` when the action returns `pendingVerification`.

### Verify-success landing

`autoSignInAfterVerification: true` is already set, so clicking the email link
signs the user in. Better Auth redirects to its `callbackURL`; point that at the
dashboard (or `/verify-email?status=success` → dashboard). Confirm the active
organization resolves correctly for a freshly-verified owner.

---

## Testing

- **Unit (vitest):**
  - `emailVerificationEnabled()` returns the env-driven boolean.
  - `registerCompany` / `acceptInviteSignup`: with key **absent** → auto-verify +
    sign-in path (asserts `emailVerified` set, `signInEmail` called); with key
    **present** → pending path (asserts no auto-verify, `pendingVerification:true`).
    Better Auth + db mocked.
  - Sentry config guard: `init` not called when DSN unset.
- **Build smoke:** `npm run build` succeeds with the new instrumentation files —
  this is the primary risk for approach B and must pass before landing.
- **Seed:** `npm run db:seed` still produces logged-in-capable accounts (no key →
  auto-verify preserved).

## Out of scope

- Password reset / forgot-password flow.
- Sentry source-map upload (`withSentryConfig`) and automatic performance tracing.
- Email-template polish beyond the existing plain-HTML messages.

## Follow-ups (post-Phase-0)

- Graduate Sentry to source-map upload once proven in production.
- Real verification-email template design.
- Password reset (its own spec).
