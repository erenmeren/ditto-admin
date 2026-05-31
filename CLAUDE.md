@AGENTS.md

# Ditto Admin

Multi-tenant admin console for **Ditto**, a digital-receipt SaaS. Kiosks replace
paper receipts with a QR code customers scan to download a digital receipt.

## Stack

- **Next.js 16** (App Router) + React 19 + TypeScript (strict), `@/*` → repo root
- **Tailwind v4** + **shadcn/ui** (style `radix-nova` — see Gotchas) + lucide-react
- **recharts** charts · **next-themes** light/dark
- **Neon** (serverless Postgres) + **Drizzle ORM** over `neon-http`
- **Better Auth** (email/password + organization plugin) — `organization = tenant`
- **Cloudflare R2** (S3-compatible) for private receipt image storage

## Commands

```bash
npm run dev          # next dev (http://localhost:3000)
npm run build        # next build
npm run db:generate  # drizzle-kit generate (SQL from lib/db/schema.ts)
npm run db:migrate   # apply migrations to Neon
npm run db:push      # push schema without a migration file
npm run db:studio    # Drizzle Studio
npm run db:seed      # seed sample data (idempotent)
npm run auth:generate # regenerate Better Auth tables → lib/db/auth-schema.generated.ts (parity check)
```

## Environment (`.env.local`, validated by `lib/env.ts`)

| Var | Purpose |
|---|---|
| `DATABASE_URL` | Neon pooled connection string |
| `BETTER_AUTH_SECRET` | Better Auth signing secret (`openssl rand -base64 32`) |
| `BETTER_AUTH_URL` | App base URL; also the prefix of public receipt links |
| `R2_ACCOUNT_ID` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` / `R2_BUCKET` | Cloudflare R2 |

`.env.example` is the committed template. CLI scripts (seed, drizzle.config) load
env via `lib/db/load-env.ts` — **import it FIRST**, before any module that reads
env at load time (ESM imports are hoisted, so an inline `dotenv` call runs too late).

## Data model (`lib/db/schema.ts`)

Better Auth core: `user` (+`role`), `session` (+`activeOrganizationId`), `account`,
`verification`. Org plugin: `organization`, `member`, `invitation`.
App tables (all FK → `organizationId`): `tenantSettings` (PK=orgId), `store`,
`device`, `receipt`, `invoice`. Relations in `lib/db/relations.ts`.

- **`organization` = tenant.** Tenant roles (owner/admin/member) live on `member`.
- **Platform/super-admin is NOT an org membership** — it's `user.role =
  'platform_admin'` (Better Auth `additionalFields`, `input:false`).
- **Money is stored in integer cents** (`perPrintPriceCents`, `unitPriceCents`,
  `amountDueCents`); the data layer converts to dollars for the UI.
- Indexes: `receipt.token` (unique), `device.pairingCode` (unique),
  `device.deviceKeyHash`, every `organizationId`.

## Architecture

- **`lib/data.ts`** — the single data seam. Same function names/return types the
  UI always used; bodies are real Drizzle queries. Tenant-panel fns take
  `organizationId` (active tenant); super-admin fns span all orgs. DB→view-model
  conversions (cents→dollars, `lastSeenAt`→ISO `lastSeen`, status mapping,
  receipt counts/series derived from the `receipt` table) all happen here.
- **`lib/session.ts`** — `getContext()`, `requireTenant()`, `requirePlatformAdmin()`.
  Route-group layouts call these to gate access and pass session/org data to `AppShell`.
- **`middleware.ts`** — optimistic cookie gate on `/admin` + `/tenant` → `/login`.
  Fine-grained role checks run in the layouts (DB available there).
- **Auth route**: `app/api/auth/[...all]/route.ts` via `toNextJsHandler`.
  Client: `lib/auth-client.ts` (`authClient`, organization plugin).
- `next.config.ts` sets `serverExternalPackages: ["better-auth",
  "@better-auth/kysely-adapter"]` — without it the auth route 500s on a
  bun:sqlite dialect bundling error.

## Device → ingest → receipt flow

1. **Provision**: a device is seeded/created with a one-time `pairingCode`.
   `claimDevice(pairingCode, storeId)` (`lib/receipts.ts`) binds it to a store,
   issues a device key (raw key returned **once**; only its SHA-256 hash is
   stored), consumes the pairing code, sets `claimedAt`.
2. **Ingest**: `POST /api/ingest` authenticated by `Authorization: Bearer
   <deviceKey>` (NOT a user session). Looks up the device by hashed key, rejects
   unknown/paused. Accepts the rendered receipt (multipart `file`, or JSON
   `{image: base64, mimeType?, deviceId?}`). Uploads to R2 under
   `receipts/{orgId}/{receiptId}`, inserts a `receipt` row with a 40-char nanoid
   `token` + status `ready`, bumps `device.lastSeenAt`/`status`. Returns
   `{ token, url }` where `url` = `{BETTER_AUTH_URL}/r/{token}` (the device renders
   this as a QR).
3. **Public receipt**: `app/(public)/r/[token]/page.tsx` — no auth, the **token
   is the capability**. Looks up by token; if ready, renders the image via a
   **fresh short-lived presigned R2 GET URL** (`lib/storage.ts`, 5-min TTL —
   receipts are private, never public). First view flips `ready → downloaded` +
   stamps `downloadedAt` (the "receipt sent ✓" signal). Unknown token →
   graceful not-found.

## Gotchas

- **shadcn is style `radix-nova`** (`components.json`), on `radix-ui` + base-ui.
  Do NOT switch to the `base` color system — its "base-nova" components are
  react-aria based and lack `asChild`, which breaks the app.
- The sidebar's `SidebarMenuButton` renders a `Tooltip` when collapsed, so the
  app must be wrapped in `TooltipProvider` (done in `components/app-shell.tsx`).
- Don't pass lucide icon components (functions) across the server→client edge —
  nav is selected inside `AppShell` (client) by `workspace`.

## Seed accounts (`npm run db:seed`)

- Platform admin: **admin@ditto.app** / `password123`
- Tenant owner: **dana@roastwell.co** / `password123`
- Org "Roastwell Coffee": 3 stores, 6 devices (mixed status), 30 receipts, 2 invoices.
