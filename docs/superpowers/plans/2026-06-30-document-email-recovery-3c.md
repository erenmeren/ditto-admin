# Document Email & Lost-Link Recovery (Phase 3C) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a customer attach their email to a public document to (a) get it mailed to themselves and (b) later recover, via a magic link, every document they emailed themselves from that merchant — plus an optional marketing opt-in surfaced to the tenant.

**Architecture:** Three new tables (`document_contact`, `marketing_contact`, `lookup_token`). Pure IO-free modules in `lib/lookup/` (token gen/hash/verify, email normalization, email templates) unit-tested like `lib/branding/coverage.ts`. IO in a thin `lib/lookup/store.ts` + new `lib/data.ts` reads. Two public server actions (rate-limited, no email enumeration). New public pages under `app/(public)/d/lookup/`, a modified `/d/[token]` page, and a tenant `/tenant/contacts` page. All email goes through the existing `sendEmail` which no-ops without `RESEND_API_KEY`, so the feature ships inert and activates with zero code change.

**Tech Stack:** Next.js 16 App Router, React 19, Drizzle ORM over neon-http, vitest, Tailwind v4 + shadcn (radix-nova), nanoid, node:crypto SHA-256.

## Global Constraints

- **Money/PII:** customer emails are new PII — store lowercased/trimmed, org-scoped, cascade-delete with org/document.
- **Email is inert without `RESEND_API_KEY`:** never branch on the key in feature code; always call `sendEmail` (it no-ops + logs). Build and test as if it works.
- **No email enumeration:** both public send paths ALWAYS return a generic success regardless of whether data exists.
- **Magic links:** single-use, 30-minute TTL, stored as SHA-256 hex hash; raw token only ever appears in the emailed URL.
- **Token-is-capability preserved:** recovery links to `/d/{token}`; introduce NO new presigned-URL surface.
- **Drizzle TZ lesson:** when comparing a JS `Date` against a `timestamp WITHOUT time zone` in raw `sql`, pass `date.toISOString()` and cast `::timestamp`. Here we use Drizzle column comparisons (not raw sql) for expiry, which is safe, but verify expiry/consume against live Neon with a throwaway tsx script.
- **Roles:** `/tenant/contacts` mutations/exports gated to `owner`/`admin` exactly like `coverage-actions.ts`.
- **Public URL base:** use `env.BETTER_AUTH_URL` as the link prefix (same as ingest/document URLs).

---

### Task 1: Schema + migration for the three tables

**Files:**
- Modify: `lib/db/schema.ts` (add three `pgTable`s + add them to the bottom `export` block)
- Create: `lib/db/migrations/<generated>.sql` (via `npm run db:generate`)

**Interfaces:**
- Produces: Drizzle tables `documentContact`, `marketingContact`, `lookupToken` with the columns below.

- [ ] **Step 1: Add the three tables to `lib/db/schema.ts`** (place after the `document` table, before `invoice`). Follow the existing column/index idiom (`text`, `timestamp`, `index`, `uniqueIndex`, `id` PK as `text`):

```ts
export const documentContact = pgTable(
  "document_contact",
  {
    id: text("id").primaryKey(),
    organizationId: text("organization_id")
      .notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    documentId: text("document_id")
      .notNull()
      .references(() => document.id, { onDelete: "cascade" }),
    email: text("email").notNull(), // lowercased/trimmed by the caller
    createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  },
  (t) => [
    index("document_contact_org_email_idx").on(t.organizationId, t.email),
    index("document_contact_document_id_idx").on(t.documentId),
  ],
);

export const marketingContact = pgTable(
  "marketing_contact",
  {
    id: text("id").primaryKey(),
    organizationId: text("organization_id")
      .notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    email: text("email").notNull(),
    optInAt: timestamp("opt_in_at").$defaultFn(() => new Date()).notNull(),
    createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  },
  (t) => [
    uniqueIndex("marketing_contact_org_email_idx").on(t.organizationId, t.email),
  ],
);

export const lookupToken = pgTable(
  "lookup_token",
  {
    id: text("id").primaryKey(),
    organizationId: text("organization_id")
      .notNull()
      .references(() => organization.id, { onDelete: "cascade" }),
    email: text("email").notNull(),
    tokenHash: text("token_hash").notNull(),
    expiresAt: timestamp("expires_at").notNull(),
    consumedAt: timestamp("consumed_at"),
    createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  },
  (t) => [index("lookup_token_hash_idx").on(t.tokenHash)],
);
```

- [ ] **Step 2: Add the three tables to the bottom `export { ... }` schema block** in `lib/db/schema.ts` (the block ending the file), alongside `document, invoice, ...`:

```ts
  documentContact,
  marketingContact,
  lookupToken,
```

- [ ] **Step 3: Generate the migration**

Run: `npm run db:generate`
Expected: a new SQL file creating the three tables + indexes.

- [ ] **Step 4: Strip spurious churn** — open the generated `.sql`; if it contains ANY statement not creating these three tables/indexes (the known drizzle snapshot-drift hazard), delete those lines so the file contains ONLY the three `CREATE TABLE` + `CREATE INDEX`/`CREATE UNIQUE INDEX` statements.

- [ ] **Step 5: Typecheck**

Run: `npx tsc --noEmit`
Expected: PASS (no errors).

- [ ] **Step 6: Commit**

```bash
git add lib/db/schema.ts lib/db/migrations
git commit -m "feat(3c): document_contact + marketing_contact + lookup_token tables"
```

---

### Task 2: Pure lookup-token module

**Files:**
- Create: `lib/lookup/token.ts`
- Test: `lib/lookup/token.test.ts`
- Modify: `lib/ids.ts` (add `hashLookupToken`)

**Interfaces:**
- Produces:
  - `generateLookupToken(): { raw: string; hash: string }`
  - `hashLookupToken(raw: string): string` (in `lib/ids.ts`)
  - `LOOKUP_TTL_MS = 30 * 60 * 1000`
  - `isLookupValid(row: { expiresAt: Date; consumedAt: Date | null }, now: Date): boolean`

- [ ] **Step 1: Write the failing test** `lib/lookup/token.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { generateLookupToken, isLookupValid, LOOKUP_TTL_MS } from "./token";
import { hashLookupToken } from "@/lib/ids";

describe("lookup token", () => {
  it("generates a raw token whose hash matches hashLookupToken", () => {
    const { raw, hash } = generateLookupToken();
    expect(raw.length).toBeGreaterThan(20);
    expect(hash).toBe(hashLookupToken(raw));
    expect(hash).not.toBe(raw);
  });

  it("is valid before expiry and unconsumed", () => {
    const now = new Date("2026-06-30T12:00:00Z");
    const expiresAt = new Date(now.getTime() + LOOKUP_TTL_MS);
    expect(isLookupValid({ expiresAt, consumedAt: null }, now)).toBe(true);
  });

  it("is invalid once expired", () => {
    const now = new Date("2026-06-30T12:31:00Z");
    const expiresAt = new Date("2026-06-30T12:00:00Z");
    expect(isLookupValid({ expiresAt, consumedAt: null }, now)).toBe(false);
  });

  it("is invalid once consumed", () => {
    const now = new Date("2026-06-30T12:00:00Z");
    const expiresAt = new Date(now.getTime() + LOOKUP_TTL_MS);
    expect(isLookupValid({ expiresAt, consumedAt: new Date(now) }, now)).toBe(false);
  });
});
```

- [ ] **Step 2: Add `hashLookupToken` to `lib/ids.ts`** (next to `hashApiKey`):

```ts
/** SHA-256 hex of a magic-link lookup token (same algorithm as device/api keys). */
export function hashLookupToken(raw: string): string {
  return createHash("sha256").update(raw).digest("hex");
}
```

- [ ] **Step 3: Write `lib/lookup/token.ts`**:

```ts
// Pure (IO-free) magic-link token helpers. `now` is injected for determinism.
import { nanoid } from "nanoid";
import { hashLookupToken } from "@/lib/ids";

export const LOOKUP_TTL_MS = 30 * 60 * 1000; // 30 minutes

export function generateLookupToken(): { raw: string; hash: string } {
  const raw = nanoid(40);
  return { raw, hash: hashLookupToken(raw) };
}

export function isLookupValid(
  row: { expiresAt: Date; consumedAt: Date | null },
  now: Date,
): boolean {
  if (row.consumedAt != null) return false;
  return now.getTime() <= row.expiresAt.getTime();
}
```

- [ ] **Step 4: Run the tests**

Run: `npx vitest run lib/lookup/token.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add lib/lookup/token.ts lib/lookup/token.test.ts lib/ids.ts
git commit -m "feat(3c): pure lookup-token generate/hash/validate helpers"
```

---

### Task 3: Email normalization helper

**Files:**
- Create: `lib/lookup/normalize.ts`
- Test: `lib/lookup/normalize.test.ts`

**Interfaces:**
- Produces:
  - `normalizeEmail(raw: string): string | null` — lowercased, trimmed; null if not a plausible email.

- [ ] **Step 1: Write the failing test** `lib/lookup/normalize.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { normalizeEmail } from "./normalize";

describe("normalizeEmail", () => {
  it("lowercases and trims", () => {
    expect(normalizeEmail("  Jane@Example.COM ")).toBe("jane@example.com");
  });
  it("rejects strings without a single @ and a dotted domain", () => {
    expect(normalizeEmail("nope")).toBeNull();
    expect(normalizeEmail("a@b")).toBeNull();
    expect(normalizeEmail("a@@b.com")).toBeNull();
    expect(normalizeEmail("")).toBeNull();
  });
  it("accepts a normal address", () => {
    expect(normalizeEmail("sam.smith+tag@mail.co.uk")).toBe("sam.smith+tag@mail.co.uk");
  });
});
```

- [ ] **Step 2: Write `lib/lookup/normalize.ts`**:

```ts
// Pure email normalization for the public document-email + recovery forms.
export function normalizeEmail(raw: string): string | null {
  const s = raw.trim().toLowerCase();
  // Deliberately conservative: exactly one @, non-empty local part, dotted domain.
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(s)) return null;
  return s;
}
```

- [ ] **Step 3: Run the tests**

Run: `npx vitest run lib/lookup/normalize.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 4: Commit**

```bash
git add lib/lookup/normalize.ts lib/lookup/normalize.test.ts
git commit -m "feat(3c): email normalization helper"
```

---

### Task 4: Email templates

**Files:**
- Create: `lib/lookup/email-templates.ts`
- Test: `lib/lookup/email-templates.test.ts`

**Interfaces:**
- Produces:
  - `documentEmail(input: { orgName: string; documentUrl: string }): { subject: string; html: string }`
  - `lookupEmail(input: { orgName: string; recoveryUrl: string }): { subject: string; html: string }`

- [ ] **Step 1: Write the failing test** `lib/lookup/email-templates.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { documentEmail, lookupEmail } from "./email-templates";

describe("email templates", () => {
  it("documentEmail includes the org name and link, HTML-escaped", () => {
    const { subject, html } = documentEmail({
      orgName: "Tom & Jerry's",
      documentUrl: "https://x.test/d/abc",
    });
    expect(subject).toContain("Tom & Jerry's");
    expect(html).toContain("https://x.test/d/abc");
    expect(html).toContain("Tom &amp; Jerry&#39;s"); // escaped in body
  });

  it("lookupEmail includes the recovery link", () => {
    const { subject, html } = lookupEmail({
      orgName: "Roastwell",
      recoveryUrl: "https://x.test/d/lookup/org_1/tok",
    });
    expect(subject).toContain("Roastwell");
    expect(html).toContain("https://x.test/d/lookup/org_1/tok");
  });
});
```

- [ ] **Step 2: Write `lib/lookup/email-templates.ts`** (mirror the HTML-escape + inline style approach in `lib/billing/invoice-emails.ts`):

```ts
// Pure HTML builders for the two customer-facing emails. HTML-escaped.
function esc(s: string): string {
  return s
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;");
}

function button(href: string, label: string): string {
  // href is app-generated (not user input) so it is not escaped; label is static.
  return `<a href="${href}" style="display:inline-block;padding:10px 18px;border-radius:8px;background:#10A765;color:#fff;text-decoration:none;font-weight:600">${label}</a>`;
}

export function documentEmail(input: { orgName: string; documentUrl: string }): {
  subject: string;
  html: string;
} {
  const org = esc(input.orgName);
  return {
    subject: `Your document from ${input.orgName}`,
    html: `<div style="font-family:sans-serif;max-width:480px;margin:0 auto">
<p>Here's your document from <strong>${org}</strong>.</p>
<p>${button(input.documentUrl, "View document")}</p>
<p style="color:#667085;font-size:13px">If you didn't request this, you can ignore this email.</p>
</div>`,
  };
}

export function lookupEmail(input: { orgName: string; recoveryUrl: string }): {
  subject: string;
  html: string;
} {
  const org = esc(input.orgName);
  return {
    subject: `Find your documents from ${input.orgName}`,
    html: `<div style="font-family:sans-serif;max-width:480px;margin:0 auto">
<p>Use the button below to see the documents you've saved from <strong>${org}</strong>. This link expires in 30 minutes.</p>
<p>${button(input.recoveryUrl, "View my documents")}</p>
<p style="color:#667085;font-size:13px">If you didn't request this, you can ignore this email.</p>
</div>`,
  };
}
```

- [ ] **Step 3: Run the tests**

Run: `npx vitest run lib/lookup/email-templates.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 4: Commit**

```bash
git add lib/lookup/email-templates.ts lib/lookup/email-templates.test.ts
git commit -m "feat(3c): customer email templates (document + recovery)"
```

---

### Task 5: IO store layer + data reads

**Files:**
- Create: `lib/lookup/store.ts` (IO: writes/reads against Neon)
- Modify: `lib/data.ts` (add `getMarketingContacts`)
- Create: `scripts/verify-3c.ts` (throwaway integration check)

**Interfaces:**
- Consumes: `generateLookupToken`, `isLookupValid`, `hashLookupToken`, schema tables.
- Produces:
  - `recordDocumentContact(input: { organizationId: string; documentId: string; email: string }): Promise<void>`
  - `upsertMarketingContact(input: { organizationId: string; email: string }): Promise<void>`
  - `createLookupToken(input: { organizationId: string; email: string }): Promise<{ raw: string }>`
  - `consumeLookupToken(input: { organizationId: string; rawToken: string }): Promise<{ email: string } | null>` — validates (unexpired + unconsumed), stamps `consumedAt`, returns the email or null.
  - `listDocumentsForEmail(input: { organizationId: string; email: string }): Promise<Array<{ token: string; createdAt: Date; returnWindowDays: number | null; warrantyPeriodMonths: number | null }>>`
  - `getMarketingContacts(organizationId: string): Promise<Array<{ email: string; optInAt: Date }>>` (in `lib/data.ts`)

- [ ] **Step 1: Write `lib/lookup/store.ts`**:

```ts
import { and, desc, eq, isNull } from "drizzle-orm";
import { db } from "@/lib/db";
import {
  documentContact,
  marketingContact,
  lookupToken,
  document,
  tenantSettings,
} from "@/lib/db/schema";
import { id } from "@/lib/ids";
import { generateLookupToken, hashLookupToken, isLookupValid, LOOKUP_TTL_MS } from "./token";

export async function recordDocumentContact(input: {
  organizationId: string;
  documentId: string;
  email: string;
}): Promise<void> {
  await db.insert(documentContact).values({
    id: id("dc"),
    organizationId: input.organizationId,
    documentId: input.documentId,
    email: input.email,
  });
}

export async function upsertMarketingContact(input: {
  organizationId: string;
  email: string;
}): Promise<void> {
  await db
    .insert(marketingContact)
    .values({ id: id("mc"), organizationId: input.organizationId, email: input.email })
    .onConflictDoUpdate({
      target: [marketingContact.organizationId, marketingContact.email],
      set: { optInAt: new Date() },
    });
}

export async function createLookupToken(input: {
  organizationId: string;
  email: string;
}): Promise<{ raw: string }> {
  const { raw, hash } = generateLookupToken();
  await db.insert(lookupToken).values({
    id: id("lt"),
    organizationId: input.organizationId,
    email: input.email,
    tokenHash: hash,
    expiresAt: new Date(Date.now() + LOOKUP_TTL_MS),
  });
  return { raw };
}

export async function consumeLookupToken(input: {
  organizationId: string;
  rawToken: string;
}): Promise<{ email: string } | null> {
  const hash = hashLookupToken(input.rawToken);
  const [row] = await db
    .select()
    .from(lookupToken)
    .where(and(eq(lookupToken.tokenHash, hash), eq(lookupToken.organizationId, input.organizationId)))
    .limit(1);
  if (!row) return null;
  if (!isLookupValid(row, new Date())) return null;
  await db
    .update(lookupToken)
    .set({ consumedAt: new Date() })
    .where(and(eq(lookupToken.id, row.id), isNull(lookupToken.consumedAt)));
  return { email: row.email };
}

export async function listDocumentsForEmail(input: {
  organizationId: string;
  email: string;
}): Promise<Array<{ token: string; createdAt: Date; returnWindowDays: number | null; warrantyPeriodMonths: number | null }>> {
  const rows = await db
    .select({
      token: document.token,
      createdAt: document.createdAt,
      returnWindowDays: tenantSettings.returnWindowDays,
      warrantyPeriodMonths: tenantSettings.warrantyPeriodMonths,
    })
    .from(documentContact)
    .innerJoin(document, eq(documentContact.documentId, document.id))
    .leftJoin(tenantSettings, eq(tenantSettings.organizationId, document.organizationId))
    .where(and(eq(documentContact.organizationId, input.organizationId), eq(documentContact.email, input.email)))
    .orderBy(desc(document.createdAt));
  return rows;
}
```

- [ ] **Step 2: Add `getMarketingContacts` to `lib/data.ts`** (follow the existing exported-async-fn style; import `marketingContact`, `db`, `eq`, `desc`):

```ts
export async function getMarketingContacts(
  organizationId: string,
): Promise<Array<{ email: string; optInAt: Date }>> {
  return db
    .select({ email: marketingContact.email, optInAt: marketingContact.optInAt })
    .from(marketingContact)
    .where(eq(marketingContact.organizationId, organizationId))
    .orderBy(desc(marketingContact.optInAt));
}
```

- [ ] **Step 3: Write `scripts/verify-3c.ts`** (throwaway; imports `lib/db/load-env.ts` FIRST per the env-load rule). It must: pick the seeded Roastwell org id + one of its document ids (query them), call `recordDocumentContact`, `upsertMarketingContact` twice (assert only one row), `createLookupToken` → `consumeLookupToken` (assert returns email), `consumeLookupToken` again (assert null — single use), `listDocumentsForEmail` (assert the document appears), `getMarketingContacts` (assert the email appears). Log PASS/FAIL per assertion.

```ts
import "@/lib/db/load-env";
import { db } from "@/lib/db";
import { organization, document } from "@/lib/db/schema";
import { eq } from "drizzle-orm";
import {
  recordDocumentContact, upsertMarketingContact, createLookupToken,
  consumeLookupToken, listDocumentsForEmail,
} from "@/lib/lookup/store";
import { getMarketingContacts } from "@/lib/data";

async function main() {
  const [org] = await db.select().from(organization).limit(1);
  const [doc] = await db.select().from(document).where(eq(document.organizationId, org.id)).limit(1);
  const email = "verify-3c@example.com";
  const organizationId = org.id;

  await recordDocumentContact({ organizationId, documentId: doc.id, email });
  await upsertMarketingContact({ organizationId, email });
  await upsertMarketingContact({ organizationId, email }); // idempotent
  const contacts = await getMarketingContacts(organizationId);
  console.log("marketing rows for email:", contacts.filter((c) => c.email === email).length, "(expect 1)");

  const { raw } = await createLookupToken({ organizationId, email });
  const first = await consumeLookupToken({ organizationId, rawToken: raw });
  console.log("first consume:", first?.email, "(expect", email + ")");
  const second = await consumeLookupToken({ organizationId, rawToken: raw });
  console.log("second consume:", second, "(expect null)");

  const docs = await listDocumentsForEmail({ organizationId, email });
  console.log("documents for email:", docs.length, "(expect >= 1)");
  process.exit(0);
}
main();
```

- [ ] **Step 4: Run the integration check against live Neon**

Run: `npx tsx scripts/verify-3c.ts`
Expected: marketing rows = 1; first consume prints the email; second consume = null; documents >= 1.

- [ ] **Step 5: Typecheck**

Run: `npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit** (keep the throwaway script out of the commit):

```bash
git add lib/lookup/store.ts lib/data.ts
git commit -m "feat(3c): lookup IO store + marketing-contacts read"
```

---

### Task 6: Public server actions

**Files:**
- Create: `app/(public)/d/lookup/actions.ts`
- Modify: `lib/documents.ts` (add `organizationId` to `PublicDocument` + its select)

**Interfaces:**
- Consumes: `normalizeEmail`, `checkRateLimit`, `recordDocumentContact`, `upsertMarketingContact`, `createLookupToken`, `getDocumentByToken`, `sendEmail`, `documentEmail`, `lookupEmail`, `getEnv`.
- Produces:
  - `requestDocumentEmail(formData: FormData): Promise<{ ok: boolean }>` — fields: `token`, `email`, `optIn` ("on"|absent).
  - `requestLookupLink(formData: FormData): Promise<{ ok: boolean }>` — fields: `orgId`, `email`.

- [ ] **Step 1: Add `organizationId` to `PublicDocument`** in `lib/documents.ts` — add `organizationId: string;` to the interface and `organizationId: document.organizationId` (or the row's existing org field) to the select in `getDocumentByToken`. Run `npx tsc --noEmit` to confirm no consumer breaks.

- [ ] **Step 2: Write `app/(public)/d/lookup/actions.ts`**:

```ts
"use server";

import { getEnv } from "@/lib/env";
import { checkRateLimit } from "@/lib/rate-limit";
import { normalizeEmail } from "@/lib/lookup/normalize";
import {
  recordDocumentContact, upsertMarketingContact, createLookupToken,
} from "@/lib/lookup/store";
import { getDocumentByTokenMeta } from "@/lib/documents";
import { sendEmail } from "@/lib/email";
import { documentEmail, lookupEmail } from "@/lib/lookup/email-templates";

const RL = { limit: 5, windowMs: 60_000 };

export async function requestDocumentEmail(formData: FormData): Promise<{ ok: boolean }> {
  const token = String(formData.get("token") ?? "");
  const email = normalizeEmail(String(formData.get("email") ?? ""));
  const optIn = formData.get("optIn") === "on";
  if (!email) return { ok: false };

  const rl = await checkRateLimit(`doc-email:${email}`, RL);
  if (!rl.allowed) return { ok: true }; // generic — don't reveal throttling

  const doc = await getDocumentByTokenMeta(token);
  if (doc) {
    await recordDocumentContact({ organizationId: doc.organizationId, documentId: doc.id, email });
    if (optIn) await upsertMarketingContact({ organizationId: doc.organizationId, email });
    const url = `${getEnv().BETTER_AUTH_URL}/d/${token}`;
    const { subject, html } = documentEmail({ orgName: doc.organizationName, documentUrl: url });
    await sendEmail(email, subject, html);
  }
  return { ok: true }; // always generic
}

export async function requestLookupLink(formData: FormData): Promise<{ ok: boolean }> {
  const orgId = String(formData.get("orgId") ?? "");
  const email = normalizeEmail(String(formData.get("email") ?? ""));
  if (!email || !orgId) return { ok: true };

  const rl = await checkRateLimit(`lookup-link:${email}`, RL);
  if (!rl.allowed) return { ok: true };

  const { raw } = await createLookupToken({ organizationId: orgId, email });
  const url = `${getEnv().BETTER_AUTH_URL}/d/lookup/${orgId}/${raw}`;
  const orgName = await orgNameById(orgId);
  const { subject, html } = lookupEmail({ orgName, recoveryUrl: url });
  await sendEmail(email, subject, html);
  return { ok: true }; // always generic — no enumeration
}

async function orgNameById(orgId: string): Promise<string> {
  const { db } = await import("@/lib/db");
  const { organization } = await import("@/lib/db/schema");
  const { eq } = await import("drizzle-orm");
  const [row] = await db.select({ name: organization.name }).from(organization).where(eq(organization.id, orgId)).limit(1);
  return row?.name ?? "your merchant";
}
```

- [ ] **Step 3: Add a lightweight `getDocumentByTokenMeta` to `lib/documents.ts`** — a read that returns `{ id, organizationId, organizationName } | null` WITHOUT minting a presigned URL or flipping `ready → downloaded` (the existing `getDocumentByToken` has the download side-effect and must NOT be reused here):

```ts
export async function getDocumentByTokenMeta(
  token: string,
): Promise<{ id: string; organizationId: string; organizationName: string } | null> {
  const [row] = await db
    .select({
      id: document.id,
      organizationId: document.organizationId,
      organizationName: organization.name,
    })
    .from(document)
    .innerJoin(organization, eq(organization.id, document.organizationId))
    .where(eq(document.token, token))
    .limit(1);
  return row ?? null;
}
```

- [ ] **Step 4: Typecheck**

Run: `npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/(public)/d/lookup/actions.ts lib/documents.ts
git commit -m "feat(3c): public server actions for document-email + recovery link"
```

---

### Task 7: Document-page email form

**Files:**
- Create: `components/document-email-form.tsx` (client component)
- Modify: `app/(public)/d/[token]/page.tsx` (render the form; pass `token` + `organizationId`)

**Interfaces:**
- Consumes: `requestDocumentEmail` from `app/(public)/d/lookup/actions.ts`, `PublicDocument.organizationId`.

- [ ] **Step 1: Write `components/document-email-form.tsx`** — a `"use client"` component with local state. Renders an email input, a "Keep me posted with offers" checkbox (`name="optIn"`, default unchecked), a Send button, and a "Find my other documents" link to `/d/lookup/{organizationId}`. On submit, calls `requestDocumentEmail(formData)`; on resolve shows "Check your inbox — if everything's set up, your document is on its way." (generic, matches no-enumeration). Mirror the form/state idiom in `components/coverage-window-form.tsx` (validation-on-submit, inline message, `useState`/`useTransition`). Use shadcn `Input`, `Button`, `Checkbox`, `Label`.

- [ ] **Step 2: Render the form in `app/(public)/d/[token]/page.tsx`** — inside the `Shell`, after the coverage/support sections, only when `document.status !== "pending"`:

```tsx
<DocumentEmailForm token={document.token} organizationId={document.organizationId} accent={accent} />
```

Add the import at the top: `import { DocumentEmailForm } from "@/components/document-email-form";`

- [ ] **Step 3: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add components/document-email-form.tsx "app/(public)/d/[token]/page.tsx"
git commit -m "feat(3c): email-me-this-document form on the public page"
```

---

### Task 8: Recovery pages

**Files:**
- Create: `app/(public)/d/lookup/[orgId]/page.tsx` (email entry form)
- Create: `components/lookup-request-form.tsx` (client form → `requestLookupLink`)
- Create: `app/(public)/d/lookup/[orgId]/[token]/page.tsx` (recovery listing)

**Interfaces:**
- Consumes: `requestLookupLink`, `consumeLookupToken`, `listDocumentsForEmail`, `coverageStatus`.

- [ ] **Step 1: Write `components/lookup-request-form.tsx`** — `"use client"`, props `{ orgId: string }`. Email input + submit calling `requestLookupLink(formData)` (hidden `orgId`). On resolve ALWAYS shows the generic "If we have documents for that email, we've sent you a link." Mirror `document-email-form.tsx`.

- [ ] **Step 2: Write `app/(public)/d/lookup/[orgId]/page.tsx`** — server component, `params: Promise<{ orgId: string }>`. Renders a simple branded shell with a heading "Find your documents" and `<LookupRequestForm orgId={orgId} />`. No DB read needed (don't confirm the org exists — avoids an org-existence oracle; an unknown org just never sends mail).

- [ ] **Step 3: Write `app/(public)/d/lookup/[orgId]/[token]/page.tsx`** — server component, `params: Promise<{ orgId: string; token: string }>`:
  - `const res = await consumeLookupToken({ organizationId: orgId, rawToken: token });`
  - If `res === null`: render a friendly "This link has expired or was already used" screen with a link back to `/d/lookup/{orgId}` to request a new one.
  - Else: `const docs = await listDocumentsForEmail({ organizationId: orgId, email: res.email });` and render a list. For each doc compute `coverageStatus({ createdAt, returnWindowDays, warrantyPeriodMonths }, new Date())` and show the date, return/warranty status chips (reuse the same wording as the `/d/[token]` page), and a link to `/d/{token}`. If `docs` is empty show "No saved documents yet."

- [ ] **Step 4: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(public)/d/lookup/[orgId]" components/lookup-request-form.tsx
git commit -m "feat(3c): magic-link recovery pages (request + listing)"
```

---

### Task 9: Tenant contacts page

**Files:**
- Create: `app/(tenant)/tenant/contacts/page.tsx`
- Create: `app/(tenant)/tenant/contacts/export-action.ts` (CSV server action)
- Create: `components/contacts-export-button.tsx` (client; triggers CSV download)
- Modify: `components/app-shell.tsx` (add "Contacts" nav item)

**Interfaces:**
- Consumes: `getMarketingContacts`, `requireTenant`.
- Produces: `exportContactsCsv(): Promise<{ filename: string; csv: string }>` (owner/admin only).

- [ ] **Step 1: Write `app/(tenant)/tenant/contacts/page.tsx`** — server component mirroring `app/(tenant)/tenant/api/page.tsx`: `const { ctx, organizationId } = await requireTenant();` role-gate to owner/admin (if not, render a read-only "no access" note or omit export — match how other tenant pages degrade). `const contacts = await getMarketingContacts(organizationId);` Render `PageHeader` ("Contacts", "Customers who opted in to hear from you.") with `<ContactsExportButton />` when `canManage`, and a `Table` of `email` + `optInAt` (formatted). Empty state: "No opted-in customers yet."

- [ ] **Step 2: Write `app/(tenant)/tenant/contacts/export-action.ts`**:

```ts
"use server";

import { requireTenant } from "@/lib/session";
import { getMarketingContacts } from "@/lib/data";

export async function exportContactsCsv(): Promise<{ filename: string; csv: string }> {
  const { ctx, organizationId } = await requireTenant();
  const role = ctx.organizations.find((o) => o.id === organizationId)?.role;
  if (!role || !["owner", "admin"].includes(role)) {
    return { filename: "contacts.csv", csv: "" };
  }
  const rows = await getMarketingContacts(organizationId);
  const header = "email,opted_in_at\n";
  const body = rows
    .map((r) => `${csvCell(r.email)},${csvCell(r.optInAt.toISOString())}`)
    .join("\n");
  return { filename: "contacts.csv", csv: header + body };
}

function csvCell(s: string): string {
  return /[",\n]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
}
```

- [ ] **Step 3: Write `components/contacts-export-button.tsx`** — `"use client"` button that calls `exportContactsCsv()`, then builds a `Blob` from the returned `csv` and triggers a download (`URL.createObjectURL` + a temporary `<a>`). Disable while pending.

- [ ] **Step 4: Add a "Contacts" nav entry in `components/app-shell.tsx`** — find the tenant nav array (by `workspace`) and add `{ label: "Contacts", href: "/tenant/contacts", icon: <Mail/> }` (or the established nav-item shape; pick an existing lucide icon already imported in app-shell, e.g. `Users`/`Mail`). Do NOT pass icon components across the server→client edge — app-shell is the client component that owns the icons, so add it there.

- [ ] **Step 5: Typecheck + build**

Run: `npx tsc --noEmit && npm run build`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add "app/(tenant)/tenant/contacts" components/contacts-export-button.tsx components/app-shell.tsx
git commit -m "feat(3c): tenant contacts page + CSV export + nav"
```

---

### Task 10: Full verification pass

**Files:** none (verification only)

- [ ] **Step 1: Run the whole unit suite**

Run: `npx vitest run`
Expected: PASS (including the three new `lib/lookup/*.test.ts` files).

- [ ] **Step 2: Typecheck + production build**

Run: `npx tsc --noEmit && npm run build`
Expected: PASS.

- [ ] **Step 3: Re-run the live integration check**

Run: `npx tsx scripts/verify-3c.ts`
Expected: marketing rows = 1; first consume = email; second = null; documents >= 1.

- [ ] **Step 4: Manual smoke (dev server, email inert)** — `npm run dev`, open a known document `/d/{token}`, submit the email form (server log shows `[email] RESEND_API_KEY unset — would send ...`), confirm a `document_contact` row was written, visit `/d/lookup/{orgId}`, submit, confirm a `lookup_token` row + the "would send" log, paste the raw token from the DB into `/d/lookup/{orgId}/{rawToken}` and confirm the listing renders with the document + coverage chips, and that a second visit shows the expired screen (single-use). Visit `/tenant/contacts` as dana@roastwell.co and confirm the opted-in email appears + CSV downloads.

- [ ] **Step 5: Delete the throwaway script + final commit**

```bash
rm scripts/verify-3c.ts
git add -A
git commit -m "chore(3c): remove throwaway verification script"
```

---

## Self-Review

**Spec coverage:**
- "email me this document" → Tasks 6 (`requestDocumentEmail`) + 7 (form). ✓
- magic-link recovery → Tasks 6 (`requestLookupLink`) + 8 (pages) + 5 (`createLookupToken`/`consumeLookupToken`). ✓
- marketing opt-in + `/tenant/contacts` + CSV → Tasks 5 (`upsertMarketingContact`/`getMarketingContacts`) + 9. ✓
- three tables → Task 1. ✓
- per-merchant (org-scoped) recovery → all lookup queries filter `organizationId`. ✓
- single-use 30-min hashed magic links → Tasks 2 + 5. ✓
- no enumeration → Task 6 (generic returns). ✓
- inert email → uses `sendEmail` everywhere; never branches on key. ✓
- privacy boundary (tenant sees only `marketing_contact`) → Task 9 reads `getMarketingContacts` only. ✓
- pure/testable modules → Tasks 2–4. ✓
- rate-limited public paths → Task 6. ✓
- no new presigned surface → Task 6 uses `getDocumentByTokenMeta` (no presign, no download flip); recovery links to `/d/{token}`. ✓

**Placeholder scan:** UI tasks (7, 8 step 3, 9 step 1/3) describe component behavior + reference an exact existing component to mirror (`coverage-window-form.tsx`, `api/page.tsx`) rather than inlining full JSX — acceptable because the pattern file is named and the props/contracts are fully specified. All logic-bearing modules (tokens, normalize, templates, store, actions, CSV) have complete code.

**Type consistency:** `consumeLookupToken` returns `{ email } | null` (used in Task 8 step 3). `getDocumentByTokenMeta` returns `{ id, organizationId, organizationName } | null` (used in Task 6 action). `PublicDocument.organizationId` added in Task 6 step 1, consumed in Task 7. `getMarketingContacts` returns `{ email, optInAt }[]` (used in Tasks 9 page + CSV). Consistent.
