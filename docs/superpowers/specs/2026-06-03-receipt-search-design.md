# Phase 2 — Receipt Search Design

_Last updated: 2026-06-03_

## Context

The `receipt` table holds rendered receipt images in R2 with structured metadata
(`organizationId`, `deviceId`, `storeId`, `token`, `status` ∈ pending/ready/downloaded,
`createdAt`, `downloadedAt`, `byteSize`, `mimeType`). There is a public render
page `app/(public)/r/[token]/page.tsx` (the token is the capability; first view
flips `ready → downloaded`), but **no staff-facing way to find a receipt**.

This feature adds a **filterable, paginated receipt list + side-effect-free
detail view** for tenants (own org) and platform admins (across orgs).

> Receipts carry **no free-text content** (no customer/amount/line items) — they
> are images. "Search" therefore means **structured filters + exact-token lookup**,
> not full-text search.

## Decisions (locked during brainstorming)

| Decision | Choice |
|---|---|
| Scope | **Tenant + platform admin** — `/tenant/receipts` (own org) and `/admin/receipts` (all orgs) |
| Detail view | **Image + metadata**, fresh presigned R2 URL, **no status flip** (staff viewing must not mark a receipt "downloaded") |
| Page size | **Fixed 25**, not user-selectable |
| Admin filters | **org + status + date + token** (store/device cross-org cascade deferred) |

## Goals

1. Tenants filter their receipts by store, device, status, date range; look up by token.
2. Platform admins filter across all orgs by org, status, date range, token.
3. Clicking a result opens a detail view showing the receipt image + metadata,
   **without** changing the receipt's status.
4. Results are paginated (25/page) with shareable filter URLs.

## Non-Goals

- Free-text search (no searchable receipt content exists).
- Cross-org store/device cascade on the admin page (deferred).
- Bulk export of filtered results (Reports already does CSV separately).
- Cursor pagination (offset is fine at this scale; revisit if needed).

---

## Architecture

### 1. Pure filter parsing (`lib/receipts-search.ts`)

```ts
export type ReceiptStatus = "pending" | "ready" | "downloaded";
export const PAGE_SIZE = 25;

export interface ReceiptFilters {
  organizationId?: string;
  storeId?: string;
  deviceId?: string;
  status?: ReceiptStatus;
  from?: Date;
  to?: Date;
  token?: string;
  page: number; // >= 1
}

/** Normalize raw URL search params into typed filters. Pure. */
export function parseReceiptFilters(raw: Record<string, string | undefined>): ReceiptFilters {
  // status validated against the enum; page clamped >= 1; from/to parsed as
  // dates (invalid → undefined); token/ids trimmed (empty → undefined).
}

/** Total pages for a result count. Pure. */
export function receiptPageCount(total: number, pageSize = PAGE_SIZE): number {
  return Math.max(1, Math.ceil(total / pageSize));
}
```

These are IO-free and unit-tested without DB.

### 2. Query layer (`lib/data.ts`)

```ts
searchReceipts(filters: ReceiptFilters): Promise<{ rows: ReceiptListRow[]; total: number }>
```

- `receipt` LEFT JOIN `store` + `device` for display names.
- `WHERE` from present filters: `eq(organizationId)` (when set), `eq(storeId)`,
  `eq(deviceId)`, `eq(status)`, `gte(createdAt, from)`, `lte(createdAt, to)`,
  `eq(token)`.
- `ORDER BY createdAt DESC`, `LIMIT PAGE_SIZE OFFSET (page-1)*PAGE_SIZE`.
- Separate `count(*)` with the same `WHERE` for `total`.
- `ReceiptListRow = { id, token, status, storeName, deviceName, createdAt, byteSize }`.

`getReceiptDetail(receiptId: string, opts: { organizationId?: string })` →
fetches one receipt (scoped to org when `organizationId` is given; else any),
joined with store/device names. Returns the row + `imageUrl` from
`presignedReceiptUrl(storageKey)` (or `null` if status `pending`). **Never writes.**

### 3. Pages + filter bar

- **`app/(tenant)/tenant/receipts/page.tsx`** — `requireTenant()`;
  `parseReceiptFilters(await searchParams)`; `searchReceipts({ organizationId, ...filters })`.
  Filter bar: store dropdown (org's stores), device dropdown (org's devices),
  status select, date from/to, token input. Results table (rows link to
  `/tenant/receipts/[id]`). Prev/Next pager.
- **`app/(admin)/admin/receipts/page.tsx`** — `requirePlatformAdmin()`;
  `searchReceipts({ ...filters })` (org optional). Filter bar: org dropdown (all
  orgs) + status + date + token. Rows link to `/admin/receipts/[id]`.
- **`components/receipts/receipt-filters.tsx`** (client) — reads current params,
  writes `?store=&device=&status=&from=&to=&token=&page=` via `router.replace`
  (resets `page` to 1 on filter change). Server page re-renders from `searchParams`.
- **`components/receipts/receipts-table.tsx`** — presentational rows + pager
  (server-rendered; the pager links carry the current filters).

### 4. Detail view

- **`app/(tenant)/tenant/receipts/[receiptId]/page.tsx`** — `requireTenant()`;
  `getReceiptDetail(id, { organizationId })`; 404 if not found/not in org.
- **`app/(admin)/admin/receipts/[receiptId]/page.tsx`** — `requirePlatformAdmin()`;
  `getReceiptDetail(id, {})`.
- **`components/receipts/receipt-detail.tsx`** — renders `<img src={imageUrl}>`
  (or "not available yet" when pending), metadata (store, device, status,
  created/downloaded times, byte size), the `token`, and a "copy public link"
  (`{BETTER_AUTH_URL}/r/{token}`). No mutations.

### 5. Nav

Add `{ label: "Receipts", href: "/tenant/receipts", icon: ReceiptText }` to
`TENANT_NAV` and `{ label: "Receipts", href: "/admin/receipts", icon: ReceiptText }`
to `ADMIN_NAV` (`lib/nav.ts`; import `ReceiptText` from lucide-react).

### 6. Data model

One additive migration: `index("receipt_store_id_idx").on(receipt.storeId)` — we
filter by store; orgId/deviceId/createdAt/token are already indexed. No column changes.

---

## Error handling

- Invalid filter values (bad status, unparseable date) are dropped by
  `parseReceiptFilters` → treated as "no filter" rather than erroring.
- `getReceiptDetail` returns null for unknown/out-of-scope ids → page renders 404
  (`notFound()`).
- Presign failure → detail page shows "image unavailable" instead of crashing.
- Empty result set → "No receipts match these filters."

## Testing

- **Pure unit (TDD):** `parseReceiptFilters` — valid/invalid status, page clamp
  (0/negative/missing → 1), date parsing (valid ISO vs garbage), token/id trim;
  `receiptPageCount` — exact multiples, remainder, zero → 1.
- **tsc + build** gate the query/page integration.
- **Manual:** filter by store/device/status/date on `/tenant/receipts`; paginate;
  token lookup; open a receipt → image renders and status is unchanged; repeat
  org-filtering on `/admin/receipts`.

## File structure

| File | Responsibility | New? |
|---|---|---|
| `lib/receipts-search.ts` | pure `parseReceiptFilters`, `receiptPageCount`, types | Create |
| `lib/receipts-search.test.ts` | tests for the pure helpers | Create |
| `lib/data.ts` | `searchReceipts`, `getReceiptDetail` | Modify |
| `lib/db/schema.ts` | + `receipt_store_id_idx` | Modify |
| `components/receipts/receipt-filters.tsx` | client filter bar (URL-driven) | Create |
| `components/receipts/receipts-table.tsx` | results table + pager | Create |
| `components/receipts/receipt-detail.tsx` | image + metadata view | Create |
| `app/(tenant)/tenant/receipts/page.tsx` | tenant list | Create |
| `app/(tenant)/tenant/receipts/[receiptId]/page.tsx` | tenant detail | Create |
| `app/(admin)/admin/receipts/page.tsx` | admin list | Create |
| `app/(admin)/admin/receipts/[receiptId]/page.tsx` | admin detail | Create |
| `lib/nav.ts` | + Receipts in both navs | Modify |

## Sequencing

1. Pure helpers + tests.
2. `receipt_store_id_idx` migration.
3. `searchReceipts` + `getReceiptDetail` data fns.
4. Shared components (filters, table, detail).
5. Tenant list + detail pages + nav.
6. Admin list + detail pages + nav.
7. Manual verification.
