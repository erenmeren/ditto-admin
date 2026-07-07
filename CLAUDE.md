@AGENTS.md

# Ditto Admin

Multi-tenant admin console for **Ditto**, a digital-document SaaS. Printers replace
paper documents with a QR code customers scan to download a digital document.

## Stack

- **Next.js 16** (App Router) + React 19 + TypeScript (strict), `@/*` → repo root
- **Tailwind v4** + **shadcn/ui** (style `radix-nova` — see Gotchas) + lucide-react
- **recharts** charts · **next-themes** light/dark
- **Neon** (serverless Postgres) + **Drizzle ORM** over `neon-http`
- **Better Auth** (email/password + organization plugin) — `organization = tenant`
- **Cloudflare R2** (S3-compatible) for private object storage (tenant branding
  assets, firmware binaries)

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
| `BETTER_AUTH_URL` | App base URL |
| `R2_ACCOUNT_ID` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` / `R2_BUCKET` | Cloudflare R2 |

`.env.example` is the committed template. CLI scripts (seed, drizzle.config) load
env via `lib/db/load-env.ts` — **import it FIRST**, before any module that reads
env at load time (ESM imports are hoisted, so an inline `dotenv` call runs too late).

## Data model (`lib/db/schema.ts`)

Better Auth core: `user` (+`role`), `session` (+`activeOrganizationId`), `account`,
`verification`. Org plugin: `organization`, `member`, `invitation`.
App tables (all FK → `organizationId`): `tenantSettings` (PK=orgId), `store`,
`device`, `document`, `invoice`. Relations in `lib/db/relations.ts`.

- **`organization` = tenant.** Tenant roles (owner/admin/member) live on `member`.
- **Platform/super-admin is NOT an org membership** — it's `user.role =
  'platform_admin'` (Better Auth `additionalFields`, `input:false`).
- **Money is stored in integer cents** (`perPrintPriceCents`, `unitPriceCents`,
  `amountDueCents`); the data layer converts to dollars for the UI.
- Indexes: `document.token` (unique), `device.pairingCode` (unique),
  `device.deviceKeyHash`, every `organizationId`.

## Architecture

- **`lib/data.ts`** — the single data seam. Same function names/return types the
  UI always used; bodies are real Drizzle queries. Tenant-panel fns take
  `organizationId` (active tenant); super-admin fns span all orgs. DB→view-model
  conversions (cents→dollars, `lastSeenAt`→ISO `lastSeen`, status mapping,
  document counts/series derived from the `document` table) all happen here.
- **`lib/session.ts`** — `getContext()`, `requireTenant()`, `requirePlatformAdmin()`.
  Route-group layouts call these to gate access and pass session/org data to `AppShell`.
- **`middleware.ts`** — optimistic cookie gate on `/admin` + `/tenant` → `/login`.
  Fine-grained role checks run in the layouts (DB available there).
- **Auth route**: `app/api/auth/[...all]/route.ts` via `toNextJsHandler`.
  Client: `lib/auth-client.ts` (`authClient`, organization plugin).
- `next.config.ts` sets `serverExternalPackages: ["better-auth",
  "@better-auth/kysely-adapter"]` — without it the auth route 500s on a
  bun:sqlite dialect bundling error.

## Device trigger flow (trigger-only model)

Ditto no longer ingests or hosts documents — customers host their own content
and pass a URL. The only device-activation path is the trigger API:

1. **Provision**: a device is seeded/created with a one-time `pairingCode`.
   `claimDevice(pairingCode, storeId)` (`lib/documents.ts`) binds it to a store,
   issues a device key (raw key returned **once**; only its SHA-256 hash is
   stored), consumes the pairing code, sets `claimedAt`.
2. **Trigger**: an authenticated caller (API key with the `devices:trigger`
   scope, plus a required `Idempotency-Key` header) does
   `POST /api/v1/devices/{deviceId}/trigger` with body
   `{ action: "show_qr", payload: { url } }` — `url` points at content the
   caller hosts themselves. `app/api/v1/devices/[deviceId]/trigger/route.ts`
   checks device ownership/online status, reserves 1 credit
   (`lib/credits.ts` `reserveCredit`, lazily reconciling expired holds first),
   and enqueues a `deviceCommand` row (`type: "trigger"`, `status: "pending"`).
3. **Poll + render + ack**: the device polls `GET /api/device/commands`
   (device-key auth) for pending commands, renders a QR from `payload.url`,
   then `POST /api/device/commands/ack` with `{ commandId, ok }`. A success ack
   settles the reserved credit (`settleHold`); a failure or expiry releases it
   (`releaseHold`).

## Gotchas

- **shadcn is style `radix-nova`** (`components.json`), on `radix-ui` + base-ui.
  Do NOT switch to the `base` color system — its "base-nova" components are
  react-aria based and lack `asChild`, which breaks the app.
- The sidebar's `SidebarMenuButton` renders a `Tooltip` when collapsed, so the
  app must be wrapped in `TooltipProvider` (done in `components/app-shell.tsx`).
- Don't pass lucide icon components (functions) across the server→client edge —
  nav is selected inside `AppShell` (client) by `workspace`.

## Seed accounts (`npm run db:seed`)

- Platform admin: **admin@ditto.app** / `123456`
- Tenant owner: **dana@roastwell.co** / `123456`
- Org "Roastwell Coffee": 3 stores, 6 claimed devices (mixed status), 3 unclaimed
  devices (with pairing codes), and a starter grant of prepaid credits.
