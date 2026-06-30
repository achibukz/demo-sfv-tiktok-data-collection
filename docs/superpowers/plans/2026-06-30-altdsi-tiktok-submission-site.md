# ALTDSI TikTok Submission Site Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Next.js submission site that accepts the two TikTok analytics CSVs produced by the companion Chrome extension, with code-by-email participant access, a typed-signature consent step, OIDC-gated admin views, and per-submission zip exports for the downstream pipeline.

**Architecture:** Single Next.js 15 App Router service running on a Linux VM at ALTDSI behind nginx, with Postgres on the same host. Participant identity is a per-creator access code stored in `iron-session` cookies; admin identity is OIDC via Auth.js. CSV validation is filename + header equality only (trust the extension for content). Soft-delete throughout; hard-delete is an explicit admin action.

**Tech Stack:** Next.js 15 (App Router) · TypeScript (strict) · Drizzle ORM · Postgres 16 · Zod · `csv-parse` · Auth.js (Generic OIDC) · `iron-session` · `nodemailer` · `rate-limiter-flexible` · `marked` + `DOMPurify` · `archiver` (zip) · vitest · `@testcontainers/postgresql` · Playwright · pnpm · Tailwind v4.

**Spec:** [docs/superpowers/specs/2026-06-30-altdsi-tiktok-submission-site-design.md](../specs/2026-06-30-altdsi-tiktok-submission-site-design.md)

---

## Pre-flight

You need locally:
- Node 20+ (verified with v24.9.0)
- pnpm 9+ (`corepack enable && corepack prepare pnpm@latest --activate`)
- Docker (for `@testcontainers/postgresql` in integration tests)
- A local Postgres for dev (`docker run -d --name sfv-pg -p 5432:5432 -e POSTGRES_PASSWORD=dev -e POSTGRES_DB=sfv postgres:16`) OR use the test container only and run dev against the same image

All commands assume the repo root: `/Users/achibukz/Code/GitHub/demo-sfv-tiktok-data-collection`.

---

## Phase 1 — Foundation (scaffold + DB schema)

### Task 1: Scaffold Next.js + TypeScript + Tailwind v4 + pnpm

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `next.config.ts`
- Create: `postcss.config.mjs`
- Create: `app/layout.tsx`
- Create: `app/globals.css`
- Create: `app/page.tsx` (placeholder, replaced in Task 17)
- Create: `.gitignore` additions
- Create: `.nvmrc`

- [ ] **Step 1: Enable pnpm via corepack**

```bash
corepack enable
corepack prepare pnpm@latest --activate
pnpm --version
```
Expected: `9.x.x` or higher.

- [ ] **Step 2: Initialize package.json**

Create `package.json`:
```json
{
  "name": "demo-sfv-tiktok-data-collection",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:smoke": "playwright test",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "tsx lib/db/migrate.ts",
    "db:studio": "drizzle-kit studio"
  },
  "engines": {
    "node": ">=20"
  }
}
```

- [ ] **Step 3: Install runtime + dev dependencies**

```bash
pnpm add next@15 react@19 react-dom@19 zod drizzle-orm pg iron-session nodemailer marked dompurify isomorphic-dompurify csv-parse rate-limiter-flexible archiver next-auth@beta @auth/drizzle-adapter
pnpm add -D typescript @types/node @types/react @types/react-dom @types/pg @types/nodemailer @types/archiver tailwindcss@next @tailwindcss/postcss postcss drizzle-kit tsx vitest @vitest/coverage-v8 @testcontainers/postgresql @playwright/test eslint eslint-config-next prettier dotenv
```

Expected: lockfile created, no errors.

- [ ] **Step 4: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noEmit": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 5: Create next.config.ts**

```ts
import type { NextConfig } from "next";

const config: NextConfig = {
  reactStrictMode: true,
  experimental: {
    serverActions: {
      bodySizeLimit: "10mb"
    }
  }
};

export default config;
```

- [ ] **Step 6: Create postcss.config.mjs**

```js
export default {
  plugins: { "@tailwindcss/postcss": {} }
};
```

- [ ] **Step 7: Create app/globals.css**

```css
@import "tailwindcss";

:root {
  --background: #ffffff;
  --foreground: #0a0a0a;
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: #0a0a0a;
    --foreground: #ededed;
  }
}

body {
  background: var(--background);
  color: var(--foreground);
  font-family: ui-sans-serif, system-ui, -apple-system, sans-serif;
}
```

- [ ] **Step 8: Create app/layout.tsx**

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "SFV Data Donation",
  description: "Submission site for the SFV research dataset.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="min-h-screen">{children}</body>
    </html>
  );
}
```

- [ ] **Step 9: Create placeholder app/page.tsx (replaced in Task 17)**

```tsx
export default function Home() {
  return (
    <main className="mx-auto max-w-md p-8">
      <h1 className="text-2xl font-semibold">SFV Data Donation</h1>
      <p className="mt-4 text-sm">Placeholder. Real landing in Task 17.</p>
    </main>
  );
}
```

- [ ] **Step 10: Extend .gitignore**

Add to the existing `.gitignore`:
```
.next/
out/
*.tsbuildinfo
next-env.d.ts
coverage/
playwright-report/
test-results/
.drizzle/
```

- [ ] **Step 11: Create .nvmrc**

```
20
```

- [ ] **Step 12: Verify build**

```bash
pnpm dev
```
Expected: server starts on `http://localhost:3000`, placeholder renders. Stop with Ctrl+C.

```bash
pnpm typecheck
```
Expected: no errors.

- [ ] **Step 13: Commit**

```bash
git add package.json pnpm-lock.yaml tsconfig.json next.config.ts postcss.config.mjs app/ .gitignore .nvmrc
git commit -m "chore: scaffold Next.js 15 + TS + Tailwind v4 + pnpm"
```

---

### Task 2: Configure ESLint + Prettier

**Files:**
- Create: `.eslintrc.json`
- Create: `.prettierrc.json`
- Create: `.prettierignore`

- [ ] **Step 1: Create .eslintrc.json**

```json
{
  "extends": ["next/core-web-vitals", "next/typescript"],
  "rules": {
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/consistent-type-imports": "error"
  }
}
```

- [ ] **Step 2: Create .prettierrc.json**

```json
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

- [ ] **Step 3: Create .prettierignore**

```
.next/
node_modules/
pnpm-lock.yaml
coverage/
drizzle/
playwright-report/
test-results/
```

- [ ] **Step 4: Run lint to verify clean**

```bash
pnpm lint
```
Expected: 0 errors, 0 warnings.

- [ ] **Step 5: Commit**

```bash
git add .eslintrc.json .prettierrc.json .prettierignore
git commit -m "chore: add eslint + prettier config"
```

---

### Task 3: Drizzle setup + .env.example + db client

**Files:**
- Create: `drizzle.config.ts`
- Create: `lib/db/client.ts`
- Create: `lib/db/migrate.ts`
- Create: `lib/env.ts`
- Create: `.env.example`
- Create: `.env.local` (gitignored)

- [ ] **Step 1: Create lib/env.ts**

```ts
import { z } from "zod";

const schema = z.object({
  DATABASE_URL: z.string().url(),
  PARTICIPANT_SESSION_SECRET: z.string().min(32),
  ADMIN_ALLOWLIST: z.string().default(""),
  OIDC_ISSUER: z.string().url().optional(),
  OIDC_CLIENT_ID: z.string().optional(),
  OIDC_CLIENT_SECRET: z.string().optional(),
  NEXTAUTH_URL: z.string().url().optional(),
  NEXTAUTH_SECRET: z.string().min(32).optional(),
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.coerce.number().int().positive().optional(),
  SMTP_USER: z.string().optional(),
  SMTP_PASS: z.string().optional(),
  MAIL_FROM: z.string().email().optional(),
  SITE_URL: z.string().url().default("http://localhost:3000"),
  RESEARCH_CONTACT_EMAIL: z.string().email().default("research@example.invalid"),
});

export const env = schema.parse(process.env);
export type Env = z.infer<typeof schema>;
```

- [ ] **Step 2: Create .env.example**

```
DATABASE_URL=postgresql://postgres:dev@localhost:5432/sfv
PARTICIPANT_SESSION_SECRET=replace-with-32-char-random-string-aaaaaaaa
ADMIN_ALLOWLIST=admin@example.com
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=replace-with-32-char-random-string-bbbbbbbb
OIDC_ISSUER=
OIDC_CLIENT_ID=
OIDC_CLIENT_SECRET=
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
MAIL_FROM=no-reply@altdsi.example
SITE_URL=http://localhost:3000
RESEARCH_CONTACT_EMAIL=research@example.invalid
```

- [ ] **Step 3: Create .env.local from .env.example with real dev values**

```bash
cp .env.example .env.local
# edit .env.local: set PARTICIPANT_SESSION_SECRET and NEXTAUTH_SECRET to 32+ random chars
```

- [ ] **Step 4: Create drizzle.config.ts**

```ts
import { defineConfig } from "drizzle-kit";
import "dotenv/config";

export default defineConfig({
  schema: "./lib/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
  strict: true,
  verbose: true,
});
```

- [ ] **Step 5: Create lib/db/client.ts**

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import { env } from "@/lib/env";
import * as schema from "./schema";

const globalForDb = globalThis as unknown as { pool?: Pool };

export const pool = globalForDb.pool ?? new Pool({ connectionString: env.DATABASE_URL });
if (process.env.NODE_ENV !== "production") globalForDb.pool = pool;

export const db = drizzle(pool, { schema });
export type Db = typeof db;
```

- [ ] **Step 6: Create lib/db/migrate.ts**

```ts
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { db, pool } from "./client";

async function main() {
  await migrate(db, { migrationsFolder: "./drizzle" });
  await pool.end();
  console.log("Migrations applied.");
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

- [ ] **Step 7: Commit**

```bash
git add lib/env.ts lib/db/client.ts lib/db/migrate.ts drizzle.config.ts .env.example
git commit -m "chore: drizzle config + env loader + db client"
```

---

### Task 4: Define Drizzle schema (all 5 tables) + first migration

**Files:**
- Create: `lib/db/schema.ts`
- Create: `drizzle/0000_initial.sql` (generated)

- [ ] **Step 1: Create lib/db/schema.ts**

```ts
import { sql } from "drizzle-orm";
import {
  pgTable,
  uuid,
  text,
  timestamp,
  integer,
  jsonb,
  inet,
  index,
  uniqueIndex,
  customType,
} from "drizzle-orm/pg-core";

const bytea = customType<{ data: Buffer; default: false }>({
  dataType() {
    return "bytea";
  },
});

export const invites = pgTable(
  "invites",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    accessCode: text("access_code").notNull(),
    creatorHandle: text("creator_handle").notNull(),
    email: text("email").notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    createdBy: text("created_by").notNull(),
    revokedAt: timestamp("revoked_at", { withTimezone: true }),
    lastUsedAt: timestamp("last_used_at", { withTimezone: true }),
  },
  (t) => [uniqueIndex("invites_access_code_uq").on(t.accessCode)],
);

export const submissions = pgTable(
  "submissions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    inviteId: uuid("invite_id")
      .notNull()
      .references(() => invites.id, { onDelete: "restrict" }),
    handleAtSubmission: text("handle_at_submission").notNull(),
    contactEmail: text("contact_email").notNull(),
    consentVersion: text("consent_version").notNull(),
    consentHash: text("consent_hash").notNull(),
    consentSignatureName: text("consent_signature_name").notNull(),
    consentAcceptedAt: timestamp("consent_accepted_at", { withTimezone: true }).notNull(),
    submittedAt: timestamp("submitted_at", { withTimezone: true }).notNull().defaultNow(),
    supersededAt: timestamp("superseded_at", { withTimezone: true }),
    withdrawnAt: timestamp("withdrawn_at", { withTimezone: true }),
    withdrawnBy: text("withdrawn_by"),
    withdrawnReason: text("withdrawn_reason"),
  },
  (t) => [
    index("submissions_invite_idx").on(t.inviteId),
    index("submissions_current_idx")
      .on(t.inviteId)
      .where(sql`superseded_at IS NULL AND withdrawn_at IS NULL`),
  ],
);

export const csvFiles = pgTable(
  "csv_files",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    submissionId: uuid("submission_id")
      .notNull()
      .references(() => submissions.id, { onDelete: "cascade" }),
    kind: text("kind", { enum: ["videos", "followers"] }).notNull(),
    originalFilename: text("original_filename").notNull(),
    sizeBytes: integer("size_bytes").notNull(),
    sha256: text("sha256").notNull(),
    rowCount: integer("row_count").notNull(),
    bytes: bytea("bytes"),
  },
  (t) => [uniqueIndex("csv_files_submission_kind_uq").on(t.submissionId, t.kind)],
);

export const adminAllowlist = pgTable("admin_allowlist", {
  email: text("email").primaryKey(),
  role: text("role").notNull().default("admin"),
  addedAt: timestamp("added_at", { withTimezone: true }).notNull().defaultNow(),
  addedBy: text("added_by").notNull(),
});

export const auditLog = pgTable(
  "audit_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    at: timestamp("at", { withTimezone: true }).notNull().defaultNow(),
    actorType: text("actor_type", { enum: ["participant", "admin", "system"] }).notNull(),
    actorId: text("actor_id").notNull(),
    event: text("event").notNull(),
    targetId: text("target_id"),
    ip: inet("ip"),
    userAgent: text("user_agent"),
    metadata: jsonb("metadata").notNull().default(sql`'{}'::jsonb`),
  },
  (t) => [
    index("audit_log_at_idx").on(t.at),
    index("audit_log_target_idx").on(t.targetId),
    index("audit_log_event_idx").on(t.event),
  ],
);

export type Invite = typeof invites.$inferSelect;
export type NewInvite = typeof invites.$inferInsert;
export type Submission = typeof submissions.$inferSelect;
export type NewSubmission = typeof submissions.$inferInsert;
export type CsvFile = typeof csvFiles.$inferSelect;
export type NewCsvFile = typeof csvFiles.$inferInsert;
export type AuditLogRow = typeof auditLog.$inferSelect;
```

- [ ] **Step 2: Generate the initial migration**

```bash
pnpm db:generate
```
Expected: a new file under `drizzle/` (e.g. `drizzle/0000_*.sql`) and a `drizzle/meta/_journal.json`.

- [ ] **Step 3: Start a local Postgres for dev**

```bash
docker run -d --name sfv-pg -p 5432:5432 -e POSTGRES_PASSWORD=dev -e POSTGRES_DB=sfv postgres:16
```
Expected: container id printed.

- [ ] **Step 4: Apply the migration**

```bash
pnpm db:migrate
```
Expected: `Migrations applied.`

- [ ] **Step 5: Verify with psql**

```bash
docker exec sfv-pg psql -U postgres -d sfv -c "\dt"
```
Expected: 5 tables listed: `admin_allowlist`, `audit_log`, `csv_files`, `invites`, `submissions`.

- [ ] **Step 6: Commit**

```bash
git add lib/db/schema.ts drizzle/
git commit -m "feat(db): initial schema for invites, submissions, csv_files, admin_allowlist, audit_log"
```

---

### Task 5: Admin allowlist seed migration

**Files:**
- Create: `lib/db/seedAllowlist.ts`
- Modify: `package.json` (add `db:seed-allowlist`)

- [ ] **Step 1: Create lib/db/seedAllowlist.ts**

```ts
import { db, pool } from "./client";
import { adminAllowlist } from "./schema";
import { env } from "@/lib/env";
import { sql } from "drizzle-orm";

async function main() {
  const emails = env.ADMIN_ALLOWLIST.split(",").map((e) => e.trim()).filter(Boolean);
  if (emails.length === 0) {
    console.log("ADMIN_ALLOWLIST empty; nothing to seed.");
    await pool.end();
    return;
  }
  for (const email of emails) {
    await db
      .insert(adminAllowlist)
      .values({ email, role: "admin", addedBy: "seed" })
      .onConflictDoNothing();
  }
  const rows = await db.select({ email: adminAllowlist.email }).from(adminAllowlist);
  console.log("Seeded allowlist. Current rows:", rows.map((r) => r.email).join(", "));
  await pool.end();
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

- [ ] **Step 2: Add db:seed-allowlist script**

In `package.json`, add to `scripts`:
```json
"db:seed-allowlist": "tsx lib/db/seedAllowlist.ts"
```

- [ ] **Step 3: Seed with a dev admin**

In `.env.local`, set `ADMIN_ALLOWLIST=admin@example.com` and:
```bash
pnpm db:seed-allowlist
```
Expected: `Seeded allowlist. Current rows: admin@example.com`.

- [ ] **Step 4: Commit**

```bash
git add lib/db/seedAllowlist.ts package.json
git commit -m "feat(db): admin allowlist seed from ADMIN_ALLOWLIST env"
```

---

### Task 6: Vitest setup + first audit logger with tests

**Files:**
- Create: `vitest.config.ts`
- Create: `vitest.integration.config.ts`
- Create: `tests/setup.ts`
- Create: `lib/audit/log.ts`
- Create: `tests/unit/audit.test.ts`

- [ ] **Step 1: Create vitest.config.ts (unit)**

```ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: {
    include: ["tests/unit/**/*.test.ts"],
    environment: "node",
    setupFiles: ["tests/setup.ts"],
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, ".") },
  },
});
```

- [ ] **Step 2: Create vitest.integration.config.ts**

```ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: {
    include: ["tests/integration/**/*.test.ts"],
    environment: "node",
    setupFiles: ["tests/setup.ts"],
    testTimeout: 60_000,
    hookTimeout: 60_000,
    pool: "forks",
    poolOptions: { forks: { singleFork: true } },
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, ".") },
  },
});
```

- [ ] **Step 3: Create tests/setup.ts**

```ts
import "dotenv/config";
```

- [ ] **Step 4: Write the failing test — tests/unit/audit.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { AUDIT_EVENTS, type AuditEvent } from "@/lib/audit/log";

describe("audit events", () => {
  it("includes every event referenced in the spec", () => {
    const required: AuditEvent[] = [
      "invite.issued",
      "invite.revoked",
      "invite.email_failed",
      "code.entered",
      "code.rejected",
      "code.locked_out",
      "consent.accepted",
      "submission.created",
      "csv.validated",
      "csv.rejected",
      "submission.superseded",
      "admin.viewed",
      "admin.downloaded",
      "admin.denied",
      "submission.withdrawn",
      "submission.purged",
      "system.error",
    ];
    for (const e of required) {
      expect(AUDIT_EVENTS).toContain(e);
    }
  });
});
```

- [ ] **Step 5: Run test, expect failure**

```bash
pnpm test tests/unit/audit.test.ts
```
Expected: FAIL — module not found.

- [ ] **Step 6: Implement lib/audit/log.ts**

```ts
import { db } from "@/lib/db/client";
import { auditLog } from "@/lib/db/schema";

export const AUDIT_EVENTS = [
  "invite.issued",
  "invite.revoked",
  "invite.email_failed",
  "code.entered",
  "code.rejected",
  "code.locked_out",
  "consent.accepted",
  "submission.created",
  "csv.validated",
  "csv.rejected",
  "submission.superseded",
  "admin.viewed",
  "admin.downloaded",
  "admin.denied",
  "submission.withdrawn",
  "submission.purged",
  "system.error",
] as const;

export type AuditEvent = (typeof AUDIT_EVENTS)[number];
export type ActorType = "participant" | "admin" | "system";

export interface AuditInput {
  event: AuditEvent;
  actorType: ActorType;
  actorId: string;
  targetId?: string | null;
  ip?: string | null;
  userAgent?: string | null;
  metadata?: Record<string, unknown>;
}

export async function logAudit(input: AuditInput): Promise<void> {
  await db.insert(auditLog).values({
    event: input.event,
    actorType: input.actorType,
    actorId: input.actorId,
    targetId: input.targetId ?? null,
    ip: input.ip ?? null,
    userAgent: input.userAgent ?? null,
    metadata: (input.metadata ?? {}) as object,
  });
}
```

- [ ] **Step 7: Run test, expect pass**

```bash
pnpm test tests/unit/audit.test.ts
```
Expected: PASS (1).

- [ ] **Step 8: Commit**

```bash
git add vitest.config.ts vitest.integration.config.ts tests/setup.ts lib/audit/log.ts tests/unit/audit.test.ts
git commit -m "feat(audit): typed event registry + log writer"
```

---

## Phase 2 — Access code system + participant session

### Task 7: Access code generation

**Files:**
- Create: `lib/code/generate.ts`
- Create: `tests/unit/code-generate.test.ts`

- [ ] **Step 1: Write the failing test — tests/unit/code-generate.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { generateAccessCode, formatAccessCode } from "@/lib/code/generate";

describe("generateAccessCode", () => {
  it("returns 12 alphanumeric chars excluding ambiguous I, l, 1, O, 0", () => {
    for (let i = 0; i < 200; i++) {
      const code = generateAccessCode();
      expect(code).toMatch(/^[A-HJ-NP-Z2-9]{12}$/);
    }
  });

  it("produces unique codes across 10k samples", () => {
    const seen = new Set<string>();
    for (let i = 0; i < 10_000; i++) seen.add(generateAccessCode());
    expect(seen.size).toBeGreaterThan(9_990);
  });
});

describe("formatAccessCode", () => {
  it("inserts dashes every 4 chars", () => {
    expect(formatAccessCode("R7K9M2NPX8WL")).toBe("R7K9-M2NP-X8WL");
  });

  it("is a no-op on already-formatted input", () => {
    expect(formatAccessCode("R7K9-M2NP-X8WL")).toBe("R7K9-M2NP-X8WL");
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/code-generate.test.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement lib/code/generate.ts**

```ts
import { randomInt } from "node:crypto";

const ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
const LENGTH = 12;

export function generateAccessCode(): string {
  let out = "";
  for (let i = 0; i < LENGTH; i++) out += ALPHABET[randomInt(ALPHABET.length)];
  return out;
}

export function formatAccessCode(raw: string): string {
  const stripped = raw.replace(/-/g, "");
  return stripped.match(/.{1,4}/g)?.join("-") ?? stripped;
}
```

- [ ] **Step 4: Run, expect pass**

```bash
pnpm test tests/unit/code-generate.test.ts
```
Expected: PASS (3).

- [ ] **Step 5: Commit**

```bash
git add lib/code/generate.ts tests/unit/code-generate.test.ts
git commit -m "feat(code): unambiguous 12-char access code generator + formatter"
```

---

### Task 8: Access code normalization + verify lookup

**Files:**
- Create: `lib/code/verify.ts`
- Create: `tests/unit/code-verify.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { normalizeAccessCode } from "@/lib/code/verify";

describe("normalizeAccessCode", () => {
  it("strips dashes, whitespace, and uppercases", () => {
    expect(normalizeAccessCode(" r7k9-m2np-x8wl ")).toBe("R7K9M2NPX8WL");
  });

  it("returns empty string for nullish/empty input", () => {
    expect(normalizeAccessCode("")).toBe("");
  });

  it("does not introduce ambiguous chars", () => {
    expect(normalizeAccessCode("R7K9-M2NP-X8WL")).toMatch(/^[A-HJ-NP-Z2-9]+$/);
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/code-verify.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/code/verify.ts**

```ts
import { db } from "@/lib/db/client";
import { invites, type Invite } from "@/lib/db/schema";
import { and, eq, isNull } from "drizzle-orm";

export function normalizeAccessCode(input: string): string {
  return input.replace(/[\s-]/g, "").toUpperCase();
}

export type VerifyResult =
  | { ok: true; invite: Invite }
  | { ok: false; reason: "not_found" | "revoked" };

export async function verifyAccessCode(input: string): Promise<VerifyResult> {
  const code = normalizeAccessCode(input);
  if (!code) return { ok: false, reason: "not_found" };
  const [row] = await db.select().from(invites).where(eq(invites.accessCode, code)).limit(1);
  if (!row) return { ok: false, reason: "not_found" };
  if (row.revokedAt) return { ok: false, reason: "revoked" };
  return { ok: true, invite: row };
}

export async function findActiveInviteById(id: string): Promise<Invite | null> {
  const [row] = await db
    .select()
    .from(invites)
    .where(and(eq(invites.id, id), isNull(invites.revokedAt)))
    .limit(1);
  return row ?? null;
}
```

- [ ] **Step 4: Run, expect pass**

```bash
pnpm test tests/unit/code-verify.test.ts
```
Expected: PASS (3).

- [ ] **Step 5: Commit**

```bash
git add lib/code/verify.ts tests/unit/code-verify.test.ts
git commit -m "feat(code): normalize + verify with revoked guard"
```

---

### Task 9: Rate-limiter (per-IP + per-code)

**Files:**
- Create: `lib/code/rateLimit.ts`
- Create: `tests/unit/code-rateLimit.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, beforeEach } from "vitest";
import { tryCodeAttempt, resetRateLimitForTests } from "@/lib/code/rateLimit";

describe("rateLimit", () => {
  beforeEach(() => resetRateLimitForTests());

  it("allows up to 5 attempts per IP, then locks out for 30 min", async () => {
    for (let i = 0; i < 5; i++) {
      expect(await tryCodeAttempt({ ip: "1.2.3.4", code: `CODE${i}` })).toEqual({ ok: true });
    }
    const sixth = await tryCodeAttempt({ ip: "1.2.3.4", code: "CODE5" });
    expect(sixth).toMatchObject({ ok: false, reason: "ip_locked" });
  });

  it("locks a code after 5 bad attempts even across IPs", async () => {
    for (let i = 0; i < 5; i++) {
      await tryCodeAttempt({ ip: `9.9.9.${i}`, code: "TARGET" });
    }
    const r = await tryCodeAttempt({ ip: "9.9.9.99", code: "TARGET" });
    expect(r).toMatchObject({ ok: false, reason: "code_locked" });
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/code-rateLimit.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/code/rateLimit.ts**

```ts
import { RateLimiterMemory } from "rate-limiter-flexible";

const IP_POINTS = 5;
const CODE_POINTS = 5;
const LOCKOUT_S = 30 * 60;

let ipLimiter = new RateLimiterMemory({ points: IP_POINTS, duration: 15 * 60, blockDuration: LOCKOUT_S });
let codeLimiter = new RateLimiterMemory({ points: CODE_POINTS, duration: 60 * 60, blockDuration: LOCKOUT_S });

export type AttemptResult =
  | { ok: true }
  | { ok: false; reason: "ip_locked" | "code_locked"; retryAfterSeconds: number };

export async function tryCodeAttempt(opts: { ip: string; code: string }): Promise<AttemptResult> {
  try {
    await ipLimiter.consume(opts.ip, 1);
  } catch (e: unknown) {
    const msBeforeNext = (e as { msBeforeNext?: number }).msBeforeNext ?? LOCKOUT_S * 1000;
    return { ok: false, reason: "ip_locked", retryAfterSeconds: Math.ceil(msBeforeNext / 1000) };
  }
  try {
    await codeLimiter.consume(opts.code, 1);
  } catch (e: unknown) {
    const msBeforeNext = (e as { msBeforeNext?: number }).msBeforeNext ?? LOCKOUT_S * 1000;
    return { ok: false, reason: "code_locked", retryAfterSeconds: Math.ceil(msBeforeNext / 1000) };
  }
  return { ok: true };
}

export function resetRateLimitForTests(): void {
  ipLimiter = new RateLimiterMemory({ points: IP_POINTS, duration: 15 * 60, blockDuration: LOCKOUT_S });
  codeLimiter = new RateLimiterMemory({ points: CODE_POINTS, duration: 60 * 60, blockDuration: LOCKOUT_S });
}
```

- [ ] **Step 4: Run, expect pass**

```bash
pnpm test tests/unit/code-rateLimit.test.ts
```
Expected: PASS (2).

- [ ] **Step 5: Commit**

```bash
git add lib/code/rateLimit.ts tests/unit/code-rateLimit.test.ts
git commit -m "feat(code): per-IP and per-code rate-limit with 30m lockout"
```

---

### Task 10: Participant session (iron-session) helpers

**Files:**
- Create: `lib/auth/participantSession.ts`
- Create: `tests/unit/participantSession.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { sealParticipantSession, unsealParticipantSession } from "@/lib/auth/participantSession";

describe("participant session", () => {
  it("roundtrips a session payload", async () => {
    const payload = {
      inviteId: "00000000-0000-0000-0000-000000000001",
      nonce: "n-1",
      startedAt: 1_700_000_000_000,
      consent: {
        version: "v1.0",
        hash: "abc",
        signatureName: "Jane Doe",
        acceptedAt: 1_700_000_000_500,
      },
    };
    const sealed = await sealParticipantSession(payload);
    const unsealed = await unsealParticipantSession(sealed);
    expect(unsealed).toEqual(payload);
  });

  it("rejects tampered ciphertext", async () => {
    const sealed = await sealParticipantSession({
      inviteId: "00000000-0000-0000-0000-000000000002",
      nonce: "n-2",
      startedAt: 1,
    });
    const tampered = sealed.slice(0, -2) + "AA";
    await expect(unsealParticipantSession(tampered)).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/participantSession.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/auth/participantSession.ts**

```ts
import { sealData, unsealData } from "iron-session";
import { env } from "@/lib/env";

export interface ParticipantSessionData {
  inviteId: string;
  nonce: string;
  startedAt: number;
  consent?: {
    version: string;
    hash: string;
    signatureName: string;
    acceptedAt: number;
  };
}

const TTL_SECONDS = 60 * 60;

export async function sealParticipantSession(data: ParticipantSessionData): Promise<string> {
  return sealData(data, { password: env.PARTICIPANT_SESSION_SECRET, ttl: TTL_SECONDS });
}

export async function unsealParticipantSession(sealed: string): Promise<ParticipantSessionData> {
  return unsealData<ParticipantSessionData>(sealed, {
    password: env.PARTICIPANT_SESSION_SECRET,
    ttl: TTL_SECONDS,
  });
}

export const PARTICIPANT_COOKIE = "sfv_p";
```

- [ ] **Step 4: Run, expect pass**

```bash
pnpm test tests/unit/participantSession.test.ts
```
Expected: PASS (2).

- [ ] **Step 5: Commit**

```bash
git add lib/auth/participantSession.ts tests/unit/participantSession.test.ts
git commit -m "feat(auth): iron-session sealed participant cookie"
```

---

## Phase 3 — CSV validation + sanitizer

### Task 11: Copy real extension fixtures into tests/fixtures

**Files:**
- Create: `tests/fixtures/tiktok_videos_unknown_2026-06-30.csv`
- Create: `tests/fixtures/tiktok_followers_unknown_2026-06-30.csv`

- [ ] **Step 1: Copy fixtures verbatim**

```bash
mkdir -p tests/fixtures
cp ~/Downloads/tiktok_videos_unknown_2026-06-30.csv tests/fixtures/
cp ~/Downloads/tiktok_followers_unknown_2026-06-30.csv tests/fixtures/
```

- [ ] **Step 2: Verify header lines**

```bash
head -1 tests/fixtures/tiktok_videos_unknown_2026-06-30.csv
head -1 tests/fixtures/tiktok_followers_unknown_2026-06-30.csv
```
Expected:
```
video_id,post_date,post_time,caption,duration_ms,comments,shares,ECR,avg_watch_time_s,NAWP,watched_full_pct,traffic_foryou_pct,traffic_follow_pct,traffic_profile_pct,traffic_search_pct,new_followers,data_quality
date,follower_count,daily_net,creator_handle,creator_uid,data_quality
```

- [ ] **Step 3: Commit**

```bash
git add tests/fixtures/
git commit -m "test: real extension CSV fixtures (videos + followers)"
```

---

### Task 12: CSV schemas + filename kind inference

**Files:**
- Create: `lib/csv/schemas.ts`
- Create: `lib/csv/kinds.ts`
- Create: `tests/unit/csv-kinds.test.ts`

- [ ] **Step 1: Create lib/csv/schemas.ts (no test — pure constants)**

```ts
export const EXPECTED_SCHEMA_VERSION = "v2026-06-30";

export const VIDEO_HEADERS = [
  "video_id",
  "post_date",
  "post_time",
  "caption",
  "duration_ms",
  "comments",
  "shares",
  "ECR",
  "avg_watch_time_s",
  "NAWP",
  "watched_full_pct",
  "traffic_foryou_pct",
  "traffic_follow_pct",
  "traffic_profile_pct",
  "traffic_search_pct",
  "new_followers",
  "data_quality",
] as const;

export const FOLLOWER_HEADERS = [
  "date",
  "follower_count",
  "daily_net",
  "creator_handle",
  "creator_uid",
  "data_quality",
] as const;

export type CsvKind = "videos" | "followers";

export const HEADERS_BY_KIND: Record<CsvKind, readonly string[]> = {
  videos: VIDEO_HEADERS,
  followers: FOLLOWER_HEADERS,
};
```

- [ ] **Step 2: Write the failing test — tests/unit/csv-kinds.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { inferKindFromFilename, isAllowedFilename } from "@/lib/csv/kinds";

describe("inferKindFromFilename", () => {
  it("accepts valid videos filename", () => {
    expect(inferKindFromFilename("tiktok_videos_aki_2026-06-30.csv")).toBe("videos");
  });
  it("accepts valid followers filename", () => {
    expect(inferKindFromFilename("tiktok_followers_aki_unknown_2026-06-30.csv")).toBe("followers");
  });
  it("rejects unknown prefix", () => {
    expect(inferKindFromFilename("tiktok_audience_aki_2026-06-30.csv")).toBeNull();
  });
  it("rejects wrong extension", () => {
    expect(inferKindFromFilename("tiktok_videos_aki_2026-06-30.txt")).toBeNull();
  });
  it("rejects missing date segment", () => {
    expect(inferKindFromFilename("tiktok_videos_aki.csv")).toBeNull();
  });
  it("rejects malformed date", () => {
    expect(inferKindFromFilename("tiktok_videos_aki_2026-6-30.csv")).toBeNull();
  });
  it("isAllowedFilename matches inferKindFromFilename", () => {
    expect(isAllowedFilename("tiktok_videos_aki_2026-06-30.csv")).toBe(true);
    expect(isAllowedFilename("nope.csv")).toBe(false);
  });
});
```

- [ ] **Step 3: Run, expect failure**

```bash
pnpm test tests/unit/csv-kinds.test.ts
```
Expected: FAIL.

- [ ] **Step 4: Implement lib/csv/kinds.ts**

```ts
import type { CsvKind } from "./schemas";

const PATTERN = /^tiktok_(videos|followers)_[A-Za-z0-9_]+_\d{4}-\d{2}-\d{2}\.csv$/;

export function inferKindFromFilename(name: string): CsvKind | null {
  const m = name.match(PATTERN);
  if (!m) return null;
  return m[1] as CsvKind;
}

export function isAllowedFilename(name: string): boolean {
  return inferKindFromFilename(name) !== null;
}
```

- [ ] **Step 5: Run, expect pass**

```bash
pnpm test tests/unit/csv-kinds.test.ts
```
Expected: PASS (7).

- [ ] **Step 6: Commit**

```bash
git add lib/csv/schemas.ts lib/csv/kinds.ts tests/unit/csv-kinds.test.ts
git commit -m "feat(csv): schema constants + filename pattern + kind inference"
```

---

### Task 13: CSV validator (filename + headers, no row checks)

**Files:**
- Create: `lib/csv/validate.ts`
- Create: `tests/unit/csv-validate.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { readFileSync } from "node:fs";
import { validateCsv } from "@/lib/csv/validate";

const videosBytes = readFileSync("tests/fixtures/tiktok_videos_unknown_2026-06-30.csv");
const followersBytes = readFileSync("tests/fixtures/tiktok_followers_unknown_2026-06-30.csv");

describe("validateCsv (fixtures)", () => {
  it("accepts the real videos fixture", () => {
    const r = validateCsv({ name: "tiktok_videos_unknown_2026-06-30.csv", bytes: videosBytes });
    expect(r.ok).toBe(true);
    if (r.ok) {
      expect(r.kind).toBe("videos");
      expect(r.rowCount).toBeGreaterThan(0);
      expect(r.sha256).toMatch(/^[0-9a-f]{64}$/);
    }
  });

  it("accepts the real followers fixture (365 day rows)", () => {
    const r = validateCsv({ name: "tiktok_followers_unknown_2026-06-30.csv", bytes: followersBytes });
    expect(r.ok).toBe(true);
    if (r.ok) {
      expect(r.kind).toBe("followers");
      expect(r.rowCount).toBe(365);
    }
  });
});

describe("validateCsv (negative)", () => {
  it("rejects filename pattern mismatch", () => {
    const r = validateCsv({ name: "videos.csv", bytes: videosBytes });
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.errors.map((e) => e.code)).toContain("filename_mismatch");
  });

  it("rejects size > 5 MB", () => {
    const big = Buffer.alloc(5 * 1024 * 1024 + 1);
    const r = validateCsv({ name: "tiktok_videos_x_2026-06-30.csv", bytes: big });
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.errors.map((e) => e.code)).toContain("file_too_large");
  });

  it("rejects header mismatch", () => {
    const lines = videosBytes.toString("utf8").split("\n");
    lines[0] = "wrong,header,row";
    const r = validateCsv({
      name: "tiktok_videos_unknown_2026-06-30.csv",
      bytes: Buffer.from(lines.join("\n"), "utf8"),
    });
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.errors.map((e) => e.code)).toContain("header_mismatch");
  });

  it("rejects non-UTF-8 bytes", () => {
    const bad = Buffer.concat([Buffer.from("video_id,post_date\n"), Buffer.from([0xff, 0xfe, 0xfd])]);
    const r = validateCsv({ name: "tiktok_videos_x_2026-06-30.csv", bytes: bad });
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.errors.map((e) => e.code)).toContain("invalid_encoding");
  });

  it("rejects parser-level garbage", () => {
    const broken = Buffer.from('a,b,c\n"unterminated,row\n', "utf8");
    const r = validateCsv({ name: "tiktok_videos_x_2026-06-30.csv", bytes: broken });
    expect(r.ok).toBe(false);
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/csv-validate.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/csv/validate.ts**

```ts
import { parse } from "csv-parse/sync";
import { createHash } from "node:crypto";
import { inferKindFromFilename } from "./kinds";
import { HEADERS_BY_KIND, type CsvKind } from "./schemas";

const MAX_BYTES = 5 * 1024 * 1024;

export interface ValidationError {
  code:
    | "filename_mismatch"
    | "file_too_large"
    | "invalid_encoding"
    | "parse_error"
    | "header_mismatch"
    | "empty_file";
  detail?: string;
}

export type ValidateResult =
  | { ok: true; kind: CsvKind; rowCount: number; sha256: string; bytes: Buffer }
  | { ok: false; errors: ValidationError[] };

function isValidUtf8(buf: Buffer): boolean {
  try {
    new TextDecoder("utf-8", { fatal: true }).decode(buf);
    return true;
  } catch {
    return false;
  }
}

export function validateCsv(file: { name: string; bytes: Buffer }): ValidateResult {
  const errors: ValidationError[] = [];

  const kind = inferKindFromFilename(file.name);
  if (!kind) errors.push({ code: "filename_mismatch", detail: file.name });

  if (file.bytes.byteLength > MAX_BYTES) {
    errors.push({ code: "file_too_large", detail: `${file.bytes.byteLength} bytes` });
  }
  if (file.bytes.byteLength === 0) errors.push({ code: "empty_file" });

  if (errors.length) return { ok: false, errors };

  if (!isValidUtf8(file.bytes)) {
    return { ok: false, errors: [{ code: "invalid_encoding" }] };
  }

  let records: string[][];
  try {
    records = parse(file.bytes, {
      bom: true,
      relax_column_count: false,
      relax_quotes: false,
      skip_empty_lines: false,
    }) as string[][];
  } catch (e) {
    return { ok: false, errors: [{ code: "parse_error", detail: (e as Error).message }] };
  }

  if (records.length === 0) return { ok: false, errors: [{ code: "empty_file" }] };

  const expected = HEADERS_BY_KIND[kind!];
  const got = records[0];
  if (got.length !== expected.length || expected.some((h, i) => got[i] !== h)) {
    return {
      ok: false,
      errors: [{ code: "header_mismatch", detail: `expected ${expected.join(",")}; got ${got.join(",")}` }],
    };
  }

  const rowCount = records.length - 1;
  const sha256 = createHash("sha256").update(file.bytes).digest("hex");
  return { ok: true, kind: kind!, rowCount, sha256, bytes: file.bytes };
}
```

- [ ] **Step 4: Run, expect pass**

```bash
pnpm test tests/unit/csv-validate.test.ts
```
Expected: PASS (7).

- [ ] **Step 5: Commit**

```bash
git add lib/csv/validate.ts tests/unit/csv-validate.test.ts
git commit -m "feat(csv): identity-only validator (filename + headers + parse)"
```

---

### Task 14: Formula-injection sanitizer (export-time)

**Files:**
- Create: `lib/csv/defensive.ts`
- Create: `tests/unit/csv-defensive.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { sanitizeCsvForExport } from "@/lib/csv/defensive";

describe("sanitizeCsvForExport", () => {
  it("prefixes formula-leading cells with a single quote", () => {
    const input = Buffer.from(
      "header\n=SUM(A1:A2)\n+1234\n-cmd\n@bad\nnormal caption\n",
      "utf8",
    );
    const out = sanitizeCsvForExport(input).toString("utf8");
    expect(out).toContain("'=SUM(A1:A2)");
    expect(out).toContain("'+1234");
    expect(out).toContain("'-cmd");
    expect(out).toContain("'@bad");
    expect(out).toContain("normal caption");
  });

  it("leaves header row untouched", () => {
    const input = Buffer.from("=A,b\nx,y\n", "utf8");
    const out = sanitizeCsvForExport(input).toString("utf8");
    expect(out.split("\n")[0]).toBe("=A,b");
  });

  it("handles quoted cells", () => {
    const input = Buffer.from('h\n"=cmd|calc"\n', "utf8");
    const out = sanitizeCsvForExport(input).toString("utf8");
    expect(out).toContain('"\'=cmd|calc"');
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/csv-defensive.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/csv/defensive.ts**

```ts
import { parse } from "csv-parse/sync";
import { stringify } from "csv-stringify/sync";

const DANGEROUS_LEAD = /^[=+\-@\t\r]/;

export function sanitizeCsvForExport(bytes: Buffer): Buffer {
  const records = parse(bytes, { bom: true, relax_column_count: true }) as string[][];
  if (records.length === 0) return bytes;
  const [header, ...rest] = records;
  const sanitizedRows = rest.map((row) =>
    row.map((cell) => (DANGEROUS_LEAD.test(cell) ? `'${cell}` : cell)),
  );
  const out = stringify([header, ...sanitizedRows]);
  return Buffer.from(out, "utf8");
}
```

- [ ] **Step 4: Install csv-stringify**

```bash
pnpm add csv-stringify
```

- [ ] **Step 5: Run, expect pass**

```bash
pnpm test tests/unit/csv-defensive.test.ts
```
Expected: PASS (3).

- [ ] **Step 6: Commit**

```bash
git add lib/csv/defensive.ts tests/unit/csv-defensive.test.ts package.json pnpm-lock.yaml
git commit -m "feat(csv): export-time formula-injection sanitizer"
```

---

## Phase 4 — Mail transport + templates

### Task 15: Mail transport interface (nodemailer or dev log)

**Files:**
- Create: `lib/mail/transport.ts`
- Create: `tests/unit/mail-transport.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect, vi } from "vitest";
import { createMailTransport } from "@/lib/mail/transport";

describe("createMailTransport", () => {
  it("returns a dev-log transport when SMTP env is missing", async () => {
    const t = createMailTransport({ host: undefined, port: undefined, user: undefined, pass: undefined, from: undefined });
    const r = await t.send({ to: "x@y.com", subject: "s", text: "hi" });
    expect(r.ok).toBe(true);
    expect(t.kind).toBe("dev-log");
  });

  it("returns an smtp transport when SMTP env is complete", () => {
    const t = createMailTransport({
      host: "smtp.example.com",
      port: 587,
      user: "u",
      pass: "p",
      from: "from@example.com",
    });
    expect(t.kind).toBe("smtp");
  });

  it("returns send failure when transport throws", async () => {
    const t = createMailTransport({ host: "smtp.example.com", port: 587, user: "u", pass: "p", from: "from@example.com" });
    // Replace internal sender to simulate failure
    (t as unknown as { _transporter?: { sendMail: () => Promise<void> } })._transporter = {
      sendMail: vi.fn(async () => {
        throw new Error("boom");
      }),
    };
    const r = await t.send({ to: "x@y.com", subject: "s", text: "hi" });
    expect(r.ok).toBe(false);
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/mail-transport.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/mail/transport.ts**

```ts
import nodemailer, { type Transporter } from "nodemailer";
import { env } from "@/lib/env";

export interface MailMessage {
  to: string;
  subject: string;
  text: string;
  html?: string;
}

export interface MailSendResult {
  ok: boolean;
  error?: string;
}

export interface MailTransport {
  kind: "smtp" | "dev-log";
  send(msg: MailMessage): Promise<MailSendResult>;
}

export interface MailConfig {
  host?: string;
  port?: number;
  user?: string;
  pass?: string;
  from?: string;
}

class SmtpTransport implements MailTransport {
  kind = "smtp" as const;
  _transporter: Transporter;
  constructor(private from: string, transporter: Transporter) {
    this._transporter = transporter;
  }
  async send(msg: MailMessage): Promise<MailSendResult> {
    try {
      await this._transporter.sendMail({ from: this.from, to: msg.to, subject: msg.subject, text: msg.text, html: msg.html });
      return { ok: true };
    } catch (e) {
      return { ok: false, error: (e as Error).message };
    }
  }
}

class DevLogTransport implements MailTransport {
  kind = "dev-log" as const;
  async send(msg: MailMessage): Promise<MailSendResult> {
    console.log("[mail:dev-log]", JSON.stringify({ to: msg.to, subject: msg.subject }, null, 2));
    console.log(msg.text);
    return { ok: true };
  }
}

export function createMailTransport(cfg: MailConfig): MailTransport {
  if (!cfg.host || !cfg.port || !cfg.user || !cfg.pass || !cfg.from) return new DevLogTransport();
  const transporter = nodemailer.createTransport({
    host: cfg.host,
    port: cfg.port,
    secure: cfg.port === 465,
    auth: { user: cfg.user, pass: cfg.pass },
  });
  return new SmtpTransport(cfg.from, transporter);
}

let cached: MailTransport | null = null;
export function getMailTransport(): MailTransport {
  if (cached) return cached;
  cached = createMailTransport({
    host: env.SMTP_HOST,
    port: env.SMTP_PORT,
    user: env.SMTP_USER,
    pass: env.SMTP_PASS,
    from: env.MAIL_FROM,
  });
  return cached;
}

export function resetMailTransportForTests(): void {
  cached = null;
}
```

- [ ] **Step 4: Run, expect pass**

```bash
pnpm test tests/unit/mail-transport.test.ts
```
Expected: PASS (3).

- [ ] **Step 5: Commit**

```bash
git add lib/mail/transport.ts tests/unit/mail-transport.test.ts
git commit -m "feat(mail): nodemailer transport with dev-log fallback"
```

---

### Task 16: Mail templates (invite, confirmation, reissue)

**Files:**
- Create: `lib/mail/templates/invite.ts`
- Create: `lib/mail/templates/confirmation.ts`
- Create: `lib/mail/templates/reissue.ts`
- Create: `tests/unit/mail-templates.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { inviteEmail } from "@/lib/mail/templates/invite";
import { confirmationEmail } from "@/lib/mail/templates/confirmation";
import { reissueEmail } from "@/lib/mail/templates/reissue";

describe("mail templates", () => {
  it("invite carries the access code, handle, site URL, and contact", () => {
    const e = inviteEmail({
      handle: "aki",
      code: "R7K9-M2NP-X8WL",
      siteUrl: "https://submit.altdsi.example",
      contactEmail: "research@example.invalid",
    });
    expect(e.subject).toMatch(/SFV/i);
    expect(e.text).toContain("aki");
    expect(e.text).toContain("R7K9-M2NP-X8WL");
    expect(e.text).toContain("https://submit.altdsi.example");
    expect(e.text).toContain("research@example.invalid");
  });

  it("confirmation carries submission id, filenames, contact", () => {
    const e = confirmationEmail({
      shortId: "abc12345",
      videoFilename: "tiktok_videos_aki_2026-06-30.csv",
      followerFilename: "tiktok_followers_aki_2026-06-30.csv",
      siteUrl: "https://submit.altdsi.example",
      contactEmail: "research@example.invalid",
    });
    expect(e.text).toContain("abc12345");
    expect(e.text).toContain("tiktok_videos_aki_2026-06-30.csv");
    expect(e.text).toContain("research@example.invalid");
  });

  it("reissue mentions the prior code is no longer valid", () => {
    const e = reissueEmail({
      handle: "aki",
      code: "NEWC-ODE0-XX22",
      siteUrl: "https://submit.altdsi.example",
      contactEmail: "research@example.invalid",
    });
    expect(e.text).toMatch(/no longer valid/i);
    expect(e.text).toContain("NEWC-ODE0-XX22");
  });
});
```

- [ ] **Step 2: Run, expect failure**

```bash
pnpm test tests/unit/mail-templates.test.ts
```
Expected: FAIL.

- [ ] **Step 3: Implement lib/mail/templates/invite.ts**

```ts
export interface InviteEmailInput {
  handle: string;
  code: string;
  siteUrl: string;
  contactEmail: string;
}

export interface RenderedEmail {
  subject: string;
  text: string;
}

export function inviteEmail(input: InviteEmailInput): RenderedEmail {
  return {
    subject: "Your SFV research data donation — access code",
    text:
      `Hi ${input.handle},\n\n` +
      `Thank you for joining the SFV research data donation study.\n\n` +
      `Your access code is:\n\n  ${input.code}\n\n` +
      `Open ${input.siteUrl} and enter the code to start your submission. ` +
      `If you have trouble, reply to this email or contact ${input.contactEmail}.\n\n` +
      `— SFV research team`,
  };
}
```

- [ ] **Step 4: Implement lib/mail/templates/confirmation.ts**

```ts
export interface ConfirmationEmailInput {
  shortId: string;
  videoFilename: string;
  followerFilename: string;
  siteUrl: string;
  contactEmail: string;
}

export interface RenderedEmail {
  subject: string;
  text: string;
}

export function confirmationEmail(input: ConfirmationEmailInput): RenderedEmail {
  return {
    subject: "Your data donation was received",
    text:
      `Your submission has been received.\n\n` +
      `Reference: ${input.shortId}\n` +
      `Files saved:\n  - ${input.videoFilename}\n  - ${input.followerFilename}\n\n` +
      `You can resubmit any time with the same access code at ${input.siteUrl}; ` +
      `the latest version will replace this one.\n\n` +
      `To withdraw your submission, email ${input.contactEmail} with this reference.\n\n` +
      `— SFV research team`,
  };
}
```

- [ ] **Step 5: Implement lib/mail/templates/reissue.ts**

```ts
export interface ReissueEmailInput {
  handle: string;
  code: string;
  siteUrl: string;
  contactEmail: string;
}

export interface RenderedEmail {
  subject: string;
  text: string;
}

export function reissueEmail(input: ReissueEmailInput): RenderedEmail {
  return {
    subject: "Your replacement access code",
    text:
      `Hi ${input.handle},\n\n` +
      `Here is a replacement access code:\n\n  ${input.code}\n\n` +
      `Your previous code is no longer valid. Open ${input.siteUrl} to continue.\n\n` +
      `Questions: ${input.contactEmail}\n\n— SFV research team`,
  };
}
```

- [ ] **Step 6: Run, expect pass**

```bash
pnpm test tests/unit/mail-templates.test.ts
```
Expected: PASS (3).

- [ ] **Step 7: Commit**

```bash
git add lib/mail/templates/ tests/unit/mail-templates.test.ts
git commit -m "feat(mail): invite, confirmation, and reissue templates"
```

---

## Phase 5 — Participant flow (consent / submit / confirm)

### Task 17: Consent file + reader

**Files:**
- Create: `consent/v1.0.md`
- Create: `lib/consent.ts`
- Create: `tests/unit/consent.test.ts`

- [ ] **Step 1: Create consent/v1.0.md (TBD body)**

```md
<!-- consent body TBD; researchers will supply the detailed informed-consent text. -->
This is a placeholder for the SFV research data donation informed-consent form.
The detailed text will be inserted here before recruitment begins.
```

- [ ] **Step 2: Write the failing test — tests/unit/consent.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { readCurrentConsent } from "@/lib/consent";

describe("readCurrentConsent", () => {
  it("returns version v1.0, the markdown bytes, and a sha256 hash", () => {
    const c = readCurrentConsent();
    expect(c.version).toBe("v1.0");
    expect(c.markdown.length).toBeGreaterThan(0);
    expect(c.hash).toMatch(/^[0-9a-f]{64}$/);
  });

  it("is stable when the file is unchanged", () => {
    const a = readCurrentConsent();
    const b = readCurrentConsent();
    expect(a.hash).toBe(b.hash);
  });
});
```

- [ ] **Step 3: Run, expect failure**

```bash
pnpm test tests/unit/consent.test.ts
```
Expected: FAIL.

- [ ] **Step 4: Implement lib/consent.ts**

```ts
import { readFileSync } from "node:fs";
import { createHash } from "node:crypto";
import { join } from "node:path";

export const CURRENT_CONSENT_VERSION = "v1.0";

export interface Consent {
  version: string;
  markdown: string;
  hash: string;
}

export function readCurrentConsent(): Consent {
  const path = join(process.cwd(), "consent", `${CURRENT_CONSENT_VERSION}.md`);
  const markdown = readFileSync(path, "utf8");
  const hash = createHash("sha256").update(markdown, "utf8").digest("hex");
  return { version: CURRENT_CONSENT_VERSION, markdown, hash };
}
```

- [ ] **Step 5: Run, expect pass**

```bash
pnpm test tests/unit/consent.test.ts
```
Expected: PASS (2).

- [ ] **Step 6: Commit**

```bash
git add consent/ lib/consent.ts tests/unit/consent.test.ts
git commit -m "feat(consent): version-stamped reader + placeholder v1.0 body"
```

---

### Task 18: Participant landing page + enterCode server action

**Files:**
- Modify: `app/page.tsx`
- Create: `app/actions/enterCode.ts`
- Create: `app/(participant)/_components/CodeForm.tsx`

- [ ] **Step 1: Create app/actions/enterCode.ts**

```ts
"use server";

import { cookies, headers } from "next/headers";
import { redirect } from "next/navigation";
import { randomUUID } from "node:crypto";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { invites } from "@/lib/db/schema";
import { verifyAccessCode } from "@/lib/code/verify";
import { tryCodeAttempt } from "@/lib/code/rateLimit";
import { sealParticipantSession, PARTICIPANT_COOKIE } from "@/lib/auth/participantSession";
import { logAudit } from "@/lib/audit/log";

export type EnterCodeState = { error: string | null };

export async function enterCodeAction(_prev: EnterCodeState, formData: FormData): Promise<EnterCodeState> {
  const raw = String(formData.get("code") ?? "");
  const h = await headers();
  const ip = h.get("x-forwarded-for")?.split(",")[0]?.trim() ?? "unknown";
  const userAgent = h.get("user-agent") ?? null;

  const rl = await tryCodeAttempt({ ip, code: raw });
  if (!rl.ok) {
    await logAudit({
      event: "code.locked_out",
      actorType: "participant",
      actorId: "anonymous",
      ip,
      userAgent,
      metadata: { reason: rl.reason },
    });
    return { error: "Too many attempts. Try again later." };
  }

  const v = await verifyAccessCode(raw);
  if (!v.ok) {
    await logAudit({
      event: "code.rejected",
      actorType: "participant",
      actorId: "anonymous",
      ip,
      userAgent,
      metadata: { reason: v.reason },
    });
    return { error: "Code not recognized." };
  }

  await db
    .update(invites)
    .set({ lastUsedAt: new Date() })
    .where(eq(invites.id, v.invite.id));

  const sealed = await sealParticipantSession({
    inviteId: v.invite.id,
    nonce: randomUUID(),
    startedAt: Date.now(),
  });
  (await cookies()).set(PARTICIPANT_COOKIE, sealed, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/",
  });

  await logAudit({
    event: "code.entered",
    actorType: "participant",
    actorId: v.invite.id,
    targetId: v.invite.id,
    ip,
    userAgent,
  });

  redirect("/consent");
}
```

- [ ] **Step 2: Create app/(participant)/_components/CodeForm.tsx**

```tsx
"use client";

import { useActionState } from "react";
import { enterCodeAction, type EnterCodeState } from "@/app/actions/enterCode";

const initial: EnterCodeState = { error: null };

export function CodeForm() {
  const [state, action, pending] = useActionState(enterCodeAction, initial);
  return (
    <form action={action} className="mt-6 space-y-4">
      <label className="block text-sm font-medium">Access code</label>
      <input
        name="code"
        autoComplete="off"
        required
        className="w-full rounded border px-3 py-2 font-mono tracking-wider"
        placeholder="XXXX-XXXX-XXXX"
      />
      {state.error ? <p className="text-sm text-red-600">{state.error}</p> : null}
      <button
        type="submit"
        disabled={pending}
        className="rounded bg-black px-4 py-2 text-sm font-medium text-white disabled:opacity-50"
      >
        {pending ? "Checking..." : "Continue"}
      </button>
    </form>
  );
}
```

- [ ] **Step 3: Replace app/page.tsx with the landing page**

```tsx
import { CodeForm } from "@/app/(participant)/_components/CodeForm";

export default function LandingPage() {
  return (
    <main className="mx-auto max-w-md p-8">
      <h1 className="text-2xl font-semibold">SFV Data Donation</h1>
      <p className="mt-3 text-sm text-neutral-600">
        Enter the access code from your invitation email to continue.
      </p>
      <CodeForm />
    </main>
  );
}
```

- [ ] **Step 4: Smoke-test in the browser**

```bash
pnpm dev
```
Open http://localhost:3000, enter a bad code, see "Code not recognized." (Stop with Ctrl+C.)

- [ ] **Step 5: Commit**

```bash
git add app/page.tsx app/actions/ app/\(participant\)/
git commit -m "feat(participant): landing page + enterCode server action"
```

---

### Task 19: Consent page + acceptConsent action with typed signature

**Files:**
- Create: `app/(participant)/consent/page.tsx`
- Create: `app/(participant)/consent/_components/ConsentForm.tsx`
- Create: `app/actions/acceptConsent.ts`

- [ ] **Step 1: Create app/actions/acceptConsent.ts**

```ts
"use server";

import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { z } from "zod";
import {
  PARTICIPANT_COOKIE,
  sealParticipantSession,
  unsealParticipantSession,
} from "@/lib/auth/participantSession";
import { readCurrentConsent } from "@/lib/consent";
import { logAudit } from "@/lib/audit/log";

const schema = z.object({
  signatureName: z.string().trim().min(1, "Type your full name."),
  agreed: z.preprocess((v) => v === "on" || v === true, z.literal(true, { message: "Tick the consent box." })),
});

export type AcceptConsentState = { error: string | null };

export async function acceptConsentAction(_prev: AcceptConsentState, formData: FormData): Promise<AcceptConsentState> {
  const parsed = schema.safeParse({
    signatureName: formData.get("signatureName"),
    agreed: formData.get("agreed"),
  });
  if (!parsed.success) return { error: parsed.error.issues[0]?.message ?? "Invalid input." };

  const jar = await cookies();
  const sealed = jar.get(PARTICIPANT_COOKIE)?.value;
  if (!sealed) redirect("/");

  let session;
  try {
    session = await unsealParticipantSession(sealed);
  } catch {
    redirect("/");
  }

  const consent = readCurrentConsent();
  const acceptedAt = Date.now();
  const next = {
    ...session,
    consent: {
      version: consent.version,
      hash: consent.hash,
      signatureName: parsed.data.signatureName,
      acceptedAt,
    },
  };
  const resealed = await sealParticipantSession(next);
  jar.set(PARTICIPANT_COOKIE, resealed, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/",
  });

  await logAudit({
    event: "consent.accepted",
    actorType: "participant",
    actorId: session.inviteId,
    targetId: session.inviteId,
    metadata: { consent_version: consent.version, consent_hash: consent.hash },
  });

  redirect("/submit");
}
```

- [ ] **Step 2: Create app/(participant)/consent/_components/ConsentForm.tsx**

```tsx
"use client";

import { useActionState, useState } from "react";
import { acceptConsentAction, type AcceptConsentState } from "@/app/actions/acceptConsent";

const initial: AcceptConsentState = { error: null };

export function ConsentForm() {
  const [state, action, pending] = useActionState(acceptConsentAction, initial);
  const [name, setName] = useState("");
  const [agreed, setAgreed] = useState(false);
  const disabled = pending || !name.trim() || !agreed;
  return (
    <form action={action} className="mt-6 space-y-4">
      <label className="block text-sm font-medium">
        Full name (typed signature)
        <input
          name="signatureName"
          value={name}
          onChange={(e) => setName(e.target.value)}
          required
          className="mt-1 w-full rounded border px-3 py-2"
          autoComplete="off"
        />
      </label>
      <label className="flex items-start gap-2 text-sm">
        <input
          type="checkbox"
          name="agreed"
          checked={agreed}
          onChange={(e) => setAgreed(e.target.checked)}
          className="mt-1"
        />
        <span>I have read the above and consent to participating in this research.</span>
      </label>
      {state.error ? <p className="text-sm text-red-600">{state.error}</p> : null}
      <button
        type="submit"
        disabled={disabled}
        className="rounded bg-black px-4 py-2 text-sm font-medium text-white disabled:opacity-50"
      >
        {pending ? "Saving..." : "Continue"}
      </button>
    </form>
  );
}
```

- [ ] **Step 3: Create app/(participant)/consent/page.tsx**

```tsx
import { marked } from "marked";
import DOMPurify from "isomorphic-dompurify";
import { readCurrentConsent } from "@/lib/consent";
import { ConsentForm } from "./_components/ConsentForm";

export default async function ConsentPage() {
  const consent = readCurrentConsent();
  const html = DOMPurify.sanitize(await marked.parse(consent.markdown));
  return (
    <main className="mx-auto max-w-2xl p-8">
      <h1 className="text-2xl font-semibold">Informed Consent ({consent.version})</h1>
      <article
        className="prose prose-sm mt-6 max-w-none"
        dangerouslySetInnerHTML={{ __html: html }}
      />
      <ConsentForm />
    </main>
  );
}
```

- [ ] **Step 4: Smoke-test in the browser**

```bash
pnpm dev
```
Visit /consent without a session → should redirect to / (because no cookie). Then go through the landing → enter a real invite code (seed one via SQL if needed), accept consent. Confirm redirect to /submit.

- [ ] **Step 5: Commit**

```bash
git add app/\(participant\)/consent/ app/actions/acceptConsent.ts
git commit -m "feat(participant): consent page + typed signature acceptance"
```

---

### Task 20: Submit page + submitForm action + DB write

**Files:**
- Create: `app/(participant)/submit/page.tsx`
- Create: `app/(participant)/submit/_components/SubmitForm.tsx`
- Create: `app/actions/submitForm.ts`
- Create: `lib/submissions/create.ts`

- [ ] **Step 1: Create lib/submissions/create.ts**

```ts
import { and, eq, isNull } from "drizzle-orm";
import type { Db } from "@/lib/db/client";
import { csvFiles, invites, submissions, type NewCsvFile, type NewSubmission } from "@/lib/db/schema";
import type { ValidateResult } from "@/lib/csv/validate";
import { logAudit } from "@/lib/audit/log";

export interface CreateSubmissionInput {
  inviteId: string;
  handle: string;
  contactEmail: string;
  consent: { version: string; hash: string; signatureName: string; acceptedAt: number };
  videos: Extract<ValidateResult, { ok: true }>;
  followers: Extract<ValidateResult, { ok: true }>;
  ip?: string | null;
  userAgent?: string | null;
}

export async function createSubmission(db: Db, input: CreateSubmissionInput): Promise<string> {
  return await db.transaction(async (tx) => {
    const [prev] = await tx
      .select({ id: submissions.id })
      .from(submissions)
      .where(
        and(
          eq(submissions.inviteId, input.inviteId),
          isNull(submissions.supersededAt),
          isNull(submissions.withdrawnAt),
        ),
      );
    if (prev) {
      await tx
        .update(submissions)
        .set({ supersededAt: new Date() })
        .where(eq(submissions.id, prev.id));
      await tx
        .update(csvFiles)
        .set({ bytes: null })
        .where(eq(csvFiles.submissionId, prev.id));
      await logAudit({
        event: "submission.superseded",
        actorType: "participant",
        actorId: input.inviteId,
        targetId: prev.id,
        ip: input.ip,
        userAgent: input.userAgent,
      });
    }

    const newRow: NewSubmission = {
      inviteId: input.inviteId,
      handleAtSubmission: input.handle,
      contactEmail: input.contactEmail,
      consentVersion: input.consent.version,
      consentHash: input.consent.hash,
      consentSignatureName: input.consent.signatureName,
      consentAcceptedAt: new Date(input.consent.acceptedAt),
    };
    const [inserted] = await tx.insert(submissions).values(newRow).returning({ id: submissions.id });

    const videosRow: NewCsvFile = {
      submissionId: inserted.id,
      kind: "videos",
      originalFilename: `tiktok_videos_${input.handle}_${new Date().toISOString().slice(0, 10)}.csv`,
      sizeBytes: input.videos.bytes.byteLength,
      sha256: input.videos.sha256,
      rowCount: input.videos.rowCount,
      bytes: input.videos.bytes,
    };
    const followersRow: NewCsvFile = {
      submissionId: inserted.id,
      kind: "followers",
      originalFilename: `tiktok_followers_${input.handle}_${new Date().toISOString().slice(0, 10)}.csv`,
      sizeBytes: input.followers.bytes.byteLength,
      sha256: input.followers.sha256,
      rowCount: input.followers.rowCount,
      bytes: input.followers.bytes,
    };
    await tx.insert(csvFiles).values([videosRow, followersRow]);

    await tx.update(invites).set({ lastUsedAt: new Date() }).where(eq(invites.id, input.inviteId));

    await logAudit({
      event: "submission.created",
      actorType: "participant",
      actorId: input.inviteId,
      targetId: inserted.id,
      ip: input.ip,
      userAgent: input.userAgent,
    });
    await logAudit({
      event: "csv.validated",
      actorType: "participant",
      actorId: input.inviteId,
      targetId: inserted.id,
      metadata: { kind: "videos", row_count: input.videos.rowCount, sha256: input.videos.sha256 },
    });
    await logAudit({
      event: "csv.validated",
      actorType: "participant",
      actorId: input.inviteId,
      targetId: inserted.id,
      metadata: { kind: "followers", row_count: input.followers.rowCount, sha256: input.followers.sha256 },
    });

    return inserted.id;
  });
}
```

- [ ] **Step 2: Create app/actions/submitForm.ts**

```ts
"use server";

import { cookies, headers } from "next/headers";
import { redirect } from "next/navigation";
import { z } from "zod";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { invites } from "@/lib/db/schema";
import {
  PARTICIPANT_COOKIE,
  unsealParticipantSession,
} from "@/lib/auth/participantSession";
import { validateCsv, type ValidationError } from "@/lib/csv/validate";
import { createSubmission } from "@/lib/submissions/create";
import { logAudit } from "@/lib/audit/log";

const fields = z.object({
  handle: z.string().trim().min(1),
  contactEmail: z.string().trim().email(),
});

export interface FieldError {
  field: "handle" | "contactEmail" | "videos" | "followers";
  message: string;
}

export type SubmitFormState = { errors: FieldError[] };

const initial: SubmitFormState = { errors: [] };
export { initial as initialSubmitState };

export async function submitFormAction(_prev: SubmitFormState, formData: FormData): Promise<SubmitFormState> {
  const h = await headers();
  const ip = h.get("x-forwarded-for")?.split(",")[0]?.trim() ?? null;
  const userAgent = h.get("user-agent") ?? null;

  const jar = await cookies();
  const sealed = jar.get(PARTICIPANT_COOKIE)?.value;
  if (!sealed) redirect("/");
  let session;
  try {
    session = await unsealParticipantSession(sealed);
  } catch {
    redirect("/");
  }
  if (!session.consent) redirect("/consent");

  const parsed = fields.safeParse({
    handle: formData.get("handle"),
    contactEmail: formData.get("contactEmail"),
  });
  const errors: FieldError[] = [];
  if (!parsed.success) {
    for (const issue of parsed.error.issues) {
      errors.push({ field: issue.path[0] as FieldError["field"], message: issue.message });
    }
  }

  const videoFile = formData.get("videos") as File | null;
  const followerFile = formData.get("followers") as File | null;
  if (!videoFile || videoFile.size === 0) errors.push({ field: "videos", message: "Video CSV is required." });
  if (!followerFile || followerFile.size === 0) errors.push({ field: "followers", message: "Follower CSV is required." });

  if (errors.length) return { errors };

  const videoBytes = Buffer.from(await videoFile!.arrayBuffer());
  const followerBytes = Buffer.from(await followerFile!.arrayBuffer());
  const videoResult = validateCsv({ name: videoFile!.name, bytes: videoBytes });
  const followerResult = validateCsv({ name: followerFile!.name, bytes: followerBytes });

  if (!videoResult.ok) {
    for (const e of videoResult.errors) errors.push({ field: "videos", message: codeToMessage(e) });
    await logAudit({
      event: "csv.rejected",
      actorType: "participant",
      actorId: session.inviteId,
      targetId: session.inviteId,
      ip,
      userAgent,
      metadata: { kind: "videos", reason_codes: videoResult.errors.map((e) => e.code) },
    });
  }
  if (!followerResult.ok) {
    for (const e of followerResult.errors) errors.push({ field: "followers", message: codeToMessage(e) });
    await logAudit({
      event: "csv.rejected",
      actorType: "participant",
      actorId: session.inviteId,
      targetId: session.inviteId,
      ip,
      userAgent,
      metadata: { kind: "followers", reason_codes: followerResult.errors.map((e) => e.code) },
    });
  }
  if (errors.length) return { errors };

  if (videoResult.kind !== "videos") {
    return { errors: [{ field: "videos", message: "Wrong file kind: expected a tiktok_videos_* CSV." }] };
  }
  if (followerResult.kind !== "followers") {
    return { errors: [{ field: "followers", message: "Wrong file kind: expected a tiktok_followers_* CSV." }] };
  }

  const [invite] = await db.select().from(invites).where(eq(invites.id, session.inviteId));
  if (!invite || invite.revokedAt) redirect("/");

  const submissionId = await createSubmission(db, {
    inviteId: session.inviteId,
    handle: parsed.data!.handle,
    contactEmail: parsed.data!.contactEmail,
    consent: session.consent,
    videos: videoResult,
    followers: followerResult,
    ip,
    userAgent,
  });

  redirect(`/confirmed/${submissionId}`);
}

function codeToMessage(e: ValidationError): string {
  switch (e.code) {
    case "filename_mismatch":
      return "Filename does not match tiktok_videos_{handle}_{YYYY-MM-DD}.csv or tiktok_followers_{handle}_{YYYY-MM-DD}.csv.";
    case "file_too_large":
      return "File exceeds the 5 MB limit.";
    case "invalid_encoding":
      return "File is not UTF-8.";
    case "parse_error":
      return "CSV could not be parsed.";
    case "header_mismatch":
      return "Header does not match the expected columns. Re-export from the latest extension.";
    case "empty_file":
      return "File is empty.";
  }
}
```

- [ ] **Step 3: Create app/(participant)/submit/_components/SubmitForm.tsx**

```tsx
"use client";

import { useActionState } from "react";
import { submitFormAction, initialSubmitState, type SubmitFormState } from "@/app/actions/submitForm";

export function SubmitForm({ defaults }: { defaults: { handle: string; contactEmail: string } }) {
  const [state, action, pending] = useActionState<SubmitFormState, FormData>(submitFormAction, initialSubmitState);
  const errFor = (field: string) => state.errors.find((e) => e.field === field)?.message;
  return (
    <form action={action} encType="multipart/form-data" className="mt-6 space-y-4">
      <label className="block text-sm font-medium">
        TikTok handle
        <input
          name="handle"
          defaultValue={defaults.handle}
          required
          className="mt-1 w-full rounded border px-3 py-2"
        />
        {errFor("handle") ? <p className="mt-1 text-sm text-red-600">{errFor("handle")}</p> : null}
      </label>
      <label className="block text-sm font-medium">
        Contact email
        <input
          name="contactEmail"
          type="email"
          defaultValue={defaults.contactEmail}
          required
          className="mt-1 w-full rounded border px-3 py-2"
        />
        {errFor("contactEmail") ? <p className="mt-1 text-sm text-red-600">{errFor("contactEmail")}</p> : null}
      </label>
      <label className="block text-sm font-medium">
        Video analytics CSV
        <input name="videos" type="file" accept=".csv,text/csv" required className="mt-1 block w-full text-sm" />
        {errFor("videos") ? <p className="mt-1 text-sm text-red-600">{errFor("videos")}</p> : null}
      </label>
      <label className="block text-sm font-medium">
        Follower history CSV
        <input name="followers" type="file" accept=".csv,text/csv" required className="mt-1 block w-full text-sm" />
        {errFor("followers") ? <p className="mt-1 text-sm text-red-600">{errFor("followers")}</p> : null}
      </label>
      <button
        type="submit"
        disabled={pending}
        className="rounded bg-black px-4 py-2 text-sm font-medium text-white disabled:opacity-50"
      >
        {pending ? "Submitting..." : "Submit"}
      </button>
    </form>
  );
}
```

- [ ] **Step 4: Create app/(participant)/submit/page.tsx**

```tsx
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { invites } from "@/lib/db/schema";
import {
  PARTICIPANT_COOKIE,
  unsealParticipantSession,
} from "@/lib/auth/participantSession";
import { SubmitForm } from "./_components/SubmitForm";

export default async function SubmitPage() {
  const jar = await cookies();
  const sealed = jar.get(PARTICIPANT_COOKIE)?.value;
  if (!sealed) redirect("/");
  let session;
  try {
    session = await unsealParticipantSession(sealed);
  } catch {
    redirect("/");
  }
  if (!session.consent) redirect("/consent");

  const [invite] = await db.select().from(invites).where(eq(invites.id, session.inviteId));
  if (!invite || invite.revokedAt) redirect("/");

  return (
    <main className="mx-auto max-w-md p-8">
      <h1 className="text-2xl font-semibold">Submit your CSVs</h1>
      <p className="mt-3 text-sm text-neutral-600">
        Upload the two CSVs exported by the SFV TikTok Analytics Exporter.
      </p>
      <SubmitForm defaults={{ handle: invite.creatorHandle, contactEmail: invite.email }} />
    </main>
  );
}
```

- [ ] **Step 5: Commit**

```bash
git add app/\(participant\)/submit/ app/actions/submitForm.ts lib/submissions/
git commit -m "feat(participant): submit page + form action with strict CSV validation"
```

---

### Task 21: Confirmation page

**Files:**
- Create: `app/(participant)/confirmed/[id]/page.tsx`

- [ ] **Step 1: Create the page**

```tsx
import { cookies } from "next/headers";
import { redirect, notFound } from "next/navigation";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { csvFiles, submissions } from "@/lib/db/schema";
import {
  PARTICIPANT_COOKIE,
  unsealParticipantSession,
} from "@/lib/auth/participantSession";
import { env } from "@/lib/env";

export default async function ConfirmedPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;

  const jar = await cookies();
  const sealed = jar.get(PARTICIPANT_COOKIE)?.value;
  if (!sealed) redirect("/");
  let session;
  try {
    session = await unsealParticipantSession(sealed);
  } catch {
    redirect("/");
  }

  const [row] = await db.select().from(submissions).where(eq(submissions.id, id));
  if (!row || row.inviteId !== session.inviteId) notFound();

  const files = await db.select().from(csvFiles).where(eq(csvFiles.submissionId, row.id));
  const videos = files.find((f) => f.kind === "videos");
  const followers = files.find((f) => f.kind === "followers");

  const isoDate = row.submittedAt.toISOString().slice(0, 10);
  const shortId = row.id.slice(0, 8);

  return (
    <main className="mx-auto max-w-md p-8">
      <h1 className="text-2xl font-semibold">Submission received</h1>
      <dl className="mt-6 grid grid-cols-[max-content_1fr] gap-x-4 gap-y-2 text-sm">
        <dt className="text-neutral-500">Reference</dt>
        <dd className="font-mono">{shortId} <span className="text-neutral-400">({row.id})</span></dd>
        <dt className="text-neutral-500">Handle</dt>
        <dd>{row.handleAtSubmission}</dd>
        <dt className="text-neutral-500">Submitted</dt>
        <dd>{isoDate}</dd>
        <dt className="text-neutral-500">Signed</dt>
        <dd>{row.consentSignatureName}</dd>
        <dt className="text-neutral-500">Video CSV</dt>
        <dd>{videos?.originalFilename} ({videos?.rowCount} rows)</dd>
        <dt className="text-neutral-500">Follower CSV</dt>
        <dd>{followers?.originalFilename} ({followers?.rowCount} rows)</dd>
      </dl>
      <p className="mt-6 text-sm text-neutral-600">
        You can resubmit any time with the same access code; the latest version will replace this one.
      </p>
      <p className="mt-2 text-sm text-neutral-600">
        To withdraw, email <a href={`mailto:${env.RESEARCH_CONTACT_EMAIL}`} className="underline">{env.RESEARCH_CONTACT_EMAIL}</a> with the reference above.
      </p>
    </main>
  );
}
```

- [ ] **Step 2: End-to-end smoke (browser)**

```bash
pnpm dev
```
Manually walk through: code → consent → submit (with the two fixture CSVs) → confirmation. Verify reference + signed name render.

- [ ] **Step 3: Commit**

```bash
git add app/\(participant\)/confirmed/
git commit -m "feat(participant): confirmation page with signed signature display"
```

---

## Phase 6 — Admin auth (Auth.js OIDC) + middleware

### Task 22: Auth.js config + middleware + adminGuard

**Files:**
- Create: `auth.config.ts`
- Create: `auth.ts`
- Create: `app/api/auth/[...nextauth]/route.ts`
- Create: `middleware.ts`
- Create: `lib/auth/adminGuard.ts`
- Create: `tests/unit/adminGuard.test.ts`

- [ ] **Step 1: Create auth.config.ts**

```ts
import type { NextAuthConfig } from "next-auth";
import { env } from "@/lib/env";

const hasOidc =
  Boolean(env.OIDC_ISSUER) && Boolean(env.OIDC_CLIENT_ID) && Boolean(env.OIDC_CLIENT_SECRET);

export const authConfig: NextAuthConfig = {
  pages: { signIn: "/admin/login" },
  providers: hasOidc
    ? [
        {
          id: "oidc",
          name: "OIDC",
          type: "oidc",
          issuer: env.OIDC_ISSUER!,
          clientId: env.OIDC_CLIENT_ID!,
          clientSecret: env.OIDC_CLIENT_SECRET!,
        },
      ]
    : [],
  callbacks: {
    async session({ session, token }) {
      if (session.user && typeof token.email === "string") session.user.email = token.email;
      return session;
    },
  },
};

export const isOidcConfigured = hasOidc;
```

- [ ] **Step 2: Create auth.ts**

```ts
import NextAuth from "next-auth";
import { authConfig } from "./auth.config";

export const { handlers, auth, signIn, signOut } = NextAuth(authConfig);
```

- [ ] **Step 3: Create app/api/auth/[...nextauth]/route.ts**

```ts
export { GET, POST } from "@/auth";
```

Note: the export above re-exports `handlers.GET / handlers.POST`. If `next-auth@beta` requires destructuring, replace with:
```ts
import { handlers } from "@/auth";
export const { GET, POST } = handlers;
```

- [ ] **Step 4: Create middleware.ts**

```ts
import { NextResponse, type NextRequest } from "next/server";
import { auth } from "@/auth";
import { db } from "@/lib/db/client";
import { adminAllowlist } from "@/lib/db/schema";
import { eq } from "drizzle-orm";

export const config = {
  matcher: ["/admin/:path*"],
};

export default async function middleware(req: NextRequest) {
  if (req.nextUrl.pathname === "/admin/login") return NextResponse.next();
  const session = await auth();
  if (!session?.user?.email) {
    const url = new URL("/admin/login", req.url);
    return NextResponse.redirect(url);
  }
  const [ok] = await db
    .select({ email: adminAllowlist.email })
    .from(adminAllowlist)
    .where(eq(adminAllowlist.email, session.user.email))
    .limit(1);
  if (!ok) {
    const url = new URL("/admin/forbidden", req.url);
    return NextResponse.redirect(url);
  }
  const res = NextResponse.next();
  res.headers.set("x-admin-email", session.user.email);
  return res;
}
```

- [ ] **Step 5: Create lib/auth/adminGuard.ts (defense-in-depth)**

```ts
import { headers } from "next/headers";
import { redirect } from "next/navigation";
import { db } from "@/lib/db/client";
import { adminAllowlist } from "@/lib/db/schema";
import { eq } from "drizzle-orm";

export async function requireAdmin(): Promise<{ email: string }> {
  const h = await headers();
  const email = h.get("x-admin-email");
  if (!email) redirect("/admin/login");
  const [row] = await db
    .select({ email: adminAllowlist.email })
    .from(adminAllowlist)
    .where(eq(adminAllowlist.email, email))
    .limit(1);
  if (!row) redirect("/admin/forbidden");
  return { email };
}
```

- [ ] **Step 6: Write the failing test — tests/unit/adminGuard.test.ts**

```ts
import { describe, it, expect } from "vitest";
import { isOidcConfigured } from "@/auth.config";

describe("authConfig", () => {
  it("exports isOidcConfigured boolean", () => {
    expect(typeof isOidcConfigured).toBe("boolean");
  });
});
```

- [ ] **Step 7: Run, expect pass**

```bash
pnpm test tests/unit/adminGuard.test.ts
```
Expected: PASS (1). (Deeper coverage of the middleware lives in integration tests in Task 33.)

- [ ] **Step 8: Commit**

```bash
git add auth.ts auth.config.ts app/api/auth/ middleware.ts lib/auth/adminGuard.ts tests/unit/adminGuard.test.ts
git commit -m "feat(admin): Auth.js OIDC + middleware allowlist gate + requireAdmin"
```

---

### Task 23: Admin login + 503 + forbidden pages

**Files:**
- Create: `app/(admin)/admin/login/page.tsx`
- Create: `app/(admin)/admin/forbidden/page.tsx`

- [ ] **Step 1: Create app/(admin)/admin/login/page.tsx**

```tsx
import { isOidcConfigured } from "@/auth.config";
import { signIn } from "@/auth";

export default async function AdminLoginPage() {
  if (!isOidcConfigured) {
    return (
      <main className="mx-auto max-w-md p-8">
        <h1 className="text-2xl font-semibold">Admin auth not configured</h1>
        <p className="mt-3 text-sm">
          Set OIDC_ISSUER, OIDC_CLIENT_ID, OIDC_CLIENT_SECRET, NEXTAUTH_URL and NEXTAUTH_SECRET in the
          environment, then restart the server.
        </p>
      </main>
    );
  }
  return (
    <main className="mx-auto max-w-md p-8">
      <h1 className="text-2xl font-semibold">Admin sign-in</h1>
      <form
        action={async () => {
          "use server";
          await signIn("oidc", { redirectTo: "/admin" });
        }}
        className="mt-6"
      >
        <button type="submit" className="rounded bg-black px-4 py-2 text-sm font-medium text-white">
          Sign in with OIDC
        </button>
      </form>
    </main>
  );
}
```

- [ ] **Step 2: Create app/(admin)/admin/forbidden/page.tsx**

```tsx
import { auth } from "@/auth";
import { env } from "@/lib/env";

export default async function ForbiddenPage() {
  const session = await auth();
  const email = session?.user?.email ?? "(unknown)";
  return (
    <main className="mx-auto max-w-md p-8">
      <h1 className="text-2xl font-semibold">Access denied</h1>
      <p className="mt-3 text-sm">
        The email <code>{email}</code> is not on the admin allowlist for this site. If this is wrong, contact{" "}
        <a href={`mailto:${env.RESEARCH_CONTACT_EMAIL}`} className="underline">
          {env.RESEARCH_CONTACT_EMAIL}
        </a>
        .
      </p>
    </main>
  );
}
```

- [ ] **Step 3: Smoke-check the 503**

Unset `OIDC_*` in `.env.local`, restart `pnpm dev`, visit `/admin/login` → see the "not configured" page. Re-set after.

- [ ] **Step 4: Commit**

```bash
git add app/\(admin\)/admin/login/ app/\(admin\)/admin/forbidden/
git commit -m "feat(admin): login + 503 + forbidden pages"
```

---

## Phase 7 — Admin submissions

### Task 24: Admin submissions list

**Files:**
- Create: `app/(admin)/admin/page.tsx`
- Create: `app/(admin)/admin/_components/SubmissionsTable.tsx`
- Create: `lib/submissions/list.ts`

- [ ] **Step 1: Create lib/submissions/list.ts**

```ts
import { and, asc, desc, eq, isNull, isNotNull, sql } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { csvFiles, submissions } from "@/lib/db/schema";

export interface ListFilters {
  includeSuperseded: boolean;
  includeWithdrawn: boolean;
}

export interface ListRow {
  id: string;
  shortId: string;
  handle: string;
  email: string;
  submittedAt: string;
  status: "current" | "superseded" | "withdrawn";
  videoRowCount: number;
  followerRowCount: number;
}

export async function listSubmissions(filters: ListFilters): Promise<ListRow[]> {
  const rows = await db
    .select({
      id: submissions.id,
      handle: submissions.handleAtSubmission,
      email: submissions.contactEmail,
      submittedAt: submissions.submittedAt,
      supersededAt: submissions.supersededAt,
      withdrawnAt: submissions.withdrawnAt,
    })
    .from(submissions)
    .orderBy(desc(submissions.submittedAt));

  const counts = await db
    .select({
      submissionId: csvFiles.submissionId,
      kind: csvFiles.kind,
      rowCount: csvFiles.rowCount,
    })
    .from(csvFiles);
  const byId = new Map<string, { videos: number; followers: number }>();
  for (const c of counts) {
    const slot = byId.get(c.submissionId) ?? { videos: 0, followers: 0 };
    if (c.kind === "videos") slot.videos = c.rowCount;
    if (c.kind === "followers") slot.followers = c.rowCount;
    byId.set(c.submissionId, slot);
  }

  return rows
    .map((r): ListRow => {
      const status: ListRow["status"] = r.withdrawnAt
        ? "withdrawn"
        : r.supersededAt
          ? "superseded"
          : "current";
      const c = byId.get(r.id) ?? { videos: 0, followers: 0 };
      return {
        id: r.id,
        shortId: r.id.slice(0, 8),
        handle: r.handle,
        email: r.email,
        submittedAt: r.submittedAt.toISOString().slice(0, 10),
        status,
        videoRowCount: c.videos,
        followerRowCount: c.followers,
      };
    })
    .filter((r) => {
      if (r.status === "current") return true;
      if (r.status === "superseded") return filters.includeSuperseded;
      if (r.status === "withdrawn") return filters.includeWithdrawn;
      return false;
    });
}
```

- [ ] **Step 2: Create app/(admin)/admin/_components/SubmissionsTable.tsx**

```tsx
import Link from "next/link";
import type { ListRow } from "@/lib/submissions/list";

export function SubmissionsTable({ rows }: { rows: ListRow[] }) {
  if (rows.length === 0) return <p className="mt-6 text-sm text-neutral-500">No submissions yet.</p>;
  return (
    <table className="mt-6 w-full text-sm">
      <thead>
        <tr className="border-b text-left">
          <th className="py-2">Ref</th>
          <th>Handle</th>
          <th>Email</th>
          <th>Date</th>
          <th>Status</th>
          <th>Videos</th>
          <th>Followers</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        {rows.map((r) => (
          <tr key={r.id} className="border-b">
            <td className="py-2 font-mono">
              <Link href={`/admin/${r.id}`} className="underline">
                {r.shortId}
              </Link>
            </td>
            <td>{r.handle}</td>
            <td>{r.email}</td>
            <td>{r.submittedAt}</td>
            <td>
              <span className={`rounded px-2 py-0.5 text-xs ${badgeClass(r.status)}`}>{r.status}</span>
            </td>
            <td>{r.videoRowCount}</td>
            <td>{r.followerRowCount}</td>
            <td>
              <Link href={`/admin/${r.id}/download`} className="text-sm underline">
                Download
              </Link>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

function badgeClass(s: ListRow["status"]) {
  if (s === "current") return "bg-green-100 text-green-800";
  if (s === "superseded") return "bg-yellow-100 text-yellow-800";
  return "bg-red-100 text-red-800";
}
```

- [ ] **Step 3: Create app/(admin)/admin/page.tsx**

```tsx
import Link from "next/link";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { listSubmissions } from "@/lib/submissions/list";
import { SubmissionsTable } from "./_components/SubmissionsTable";

interface SearchParams {
  superseded?: string;
  withdrawn?: string;
}

export default async function AdminIndex({ searchParams }: { searchParams: Promise<SearchParams> }) {
  await requireAdmin();
  const sp = await searchParams;
  const includeSuperseded = sp.superseded === "1";
  const includeWithdrawn = sp.withdrawn === "1";
  const rows = await listSubmissions({ includeSuperseded, includeWithdrawn });

  return (
    <main className="mx-auto max-w-5xl p-8">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold">Submissions</h1>
        <Link href="/admin/download-all" className="rounded bg-black px-3 py-1.5 text-sm text-white">
          Download all current
        </Link>
      </div>
      <div className="mt-4 flex gap-4 text-sm">
        <Link
          href={{ pathname: "/admin", query: { superseded: includeSuperseded ? undefined : "1", withdrawn: includeWithdrawn ? "1" : undefined } }}
          className={includeSuperseded ? "underline" : ""}
        >
          {includeSuperseded ? "Hide" : "Show"} superseded
        </Link>
        <Link
          href={{ pathname: "/admin", query: { superseded: includeSuperseded ? "1" : undefined, withdrawn: includeWithdrawn ? undefined : "1" } }}
          className={includeWithdrawn ? "underline" : ""}
        >
          {includeWithdrawn ? "Hide" : "Show"} withdrawn
        </Link>
        <Link href="/admin/invites" className="ml-auto underline">
          Invites →
        </Link>
      </div>
      <SubmissionsTable rows={rows} />
    </main>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add app/\(admin\)/admin/page.tsx app/\(admin\)/admin/_components/ lib/submissions/list.ts
git commit -m "feat(admin): submissions list with status filters"
```

---

### Task 25: Per-submission zip generator

**Files:**
- Create: `lib/zip/perSubmission.ts`
- Create: `tests/unit/zip-perSubmission.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, it, expect } from "vitest";
import { readFileSync } from "node:fs";
import { Readable } from "node:stream";
import AdmZip from "adm-zip";
import { buildSubmissionZip } from "@/lib/zip/perSubmission";

const videosBytes = readFileSync("tests/fixtures/tiktok_videos_unknown_2026-06-30.csv");
const followersBytes = readFileSync("tests/fixtures/tiktok_followers_unknown_2026-06-30.csv");

async function collect(stream: Readable): Promise<Buffer> {
  const chunks: Buffer[] = [];
  for await (const c of stream) chunks.push(c as Buffer);
  return Buffer.concat(chunks);
}

describe("buildSubmissionZip", () => {
  it("packs the two CSVs, submission.json, consent.txt, and audit.json", async () => {
    const zip = await collect(
      buildSubmissionZip({
        submission: {
          id: "abcdef01-2345-6789-abcd-ef0123456789",
          handleAtSubmission: "aki",
          contactEmail: "aki@example.com",
          consentVersion: "v1.0",
          consentHash: "deadbeef".repeat(8),
          consentSignatureName: "Aki Bukz",
          consentAcceptedAt: new Date("2026-06-30T10:00:00Z"),
          submittedAt: new Date("2026-06-30T10:05:00Z"),
        },
        videosFile: {
          originalFilename: "tiktok_videos_unknown_2026-06-30.csv",
          bytes: videosBytes,
        },
        followersFile: {
          originalFilename: "tiktok_followers_unknown_2026-06-30.csv",
          bytes: followersBytes,
        },
        consentMarkdown: "Consent body TBD.\n",
        auditEntries: [{ at: "2026-06-30T10:05:00Z", event: "submission.created", actor: "abcdef01" }],
      }),
    );
    const az = new AdmZip(zip);
    const names = az.getEntries().map((e) => e.entryName);
    expect(names).toContain("submission_abcdef01_aki/submission.json");
    expect(names).toContain("submission_abcdef01_aki/tiktok_videos_unknown_2026-06-30.csv");
    expect(names).toContain("submission_abcdef01_aki/tiktok_followers_unknown_2026-06-30.csv");
    expect(names).toContain("submission_abcdef01_aki/consent_v1.0.txt");
    expect(names).toContain("submission_abcdef01_aki/audit.json");

    const meta = JSON.parse(az.readAsText("submission_abcdef01_aki/submission.json"));
    expect(meta.consent_signature_name).toBe("Aki Bukz");
    expect(meta.consent_hash).toBe("deadbeef".repeat(8));

    const consent = az.readAsText("submission_abcdef01_aki/consent_v1.0.txt");
    expect(consent).toContain("--- SIGNATURE ---");
    expect(consent).toContain("Signed by: Aki Bukz");
  });
});
```

- [ ] **Step 2: Install adm-zip (test only)**

```bash
pnpm add -D adm-zip @types/adm-zip
```

- [ ] **Step 3: Run, expect failure**

```bash
pnpm test tests/unit/zip-perSubmission.test.ts
```
Expected: FAIL.

- [ ] **Step 4: Implement lib/zip/perSubmission.ts**

```ts
import archiver from "archiver";
import { Readable } from "node:stream";
import { sanitizeCsvForExport } from "@/lib/csv/defensive";

export interface SubmissionZipInput {
  submission: {
    id: string;
    handleAtSubmission: string;
    contactEmail: string;
    consentVersion: string;
    consentHash: string;
    consentSignatureName: string;
    consentAcceptedAt: Date;
    submittedAt: Date;
  };
  videosFile: { originalFilename: string; bytes: Buffer };
  followersFile: { originalFilename: string; bytes: Buffer };
  consentMarkdown: string;
  auditEntries: unknown[];
}

export function buildSubmissionZip(input: SubmissionZipInput): Readable {
  const folder = `submission_${input.submission.id.slice(0, 8)}_${slug(input.submission.handleAtSubmission)}`;
  const archive = archiver("zip", { zlib: { level: 9 } });

  archive.append(JSON.stringify(buildMetaJson(input), null, 2), {
    name: `${folder}/submission.json`,
  });
  archive.append(sanitizeCsvForExport(input.videosFile.bytes), {
    name: `${folder}/${input.videosFile.originalFilename}`,
  });
  archive.append(sanitizeCsvForExport(input.followersFile.bytes), {
    name: `${folder}/${input.followersFile.originalFilename}`,
  });
  archive.append(buildConsentText(input), {
    name: `${folder}/consent_${input.submission.consentVersion}.txt`,
  });
  archive.append(JSON.stringify(input.auditEntries, null, 2), {
    name: `${folder}/audit.json`,
  });
  archive.append(EXPORT_NOTES, { name: `${folder}/EXPORT-NOTES.txt` });
  archive.finalize();
  return archive;
}

function buildMetaJson(input: SubmissionZipInput): Record<string, unknown> {
  return {
    submission_id: input.submission.id,
    handle: input.submission.handleAtSubmission,
    contact_email: input.submission.contactEmail,
    consent_version: input.submission.consentVersion,
    consent_hash: input.submission.consentHash,
    consent_signature_name: input.submission.consentSignatureName,
    consent_accepted_at: input.submission.consentAcceptedAt.toISOString(),
    submitted_at: input.submission.submittedAt.toISOString(),
  };
}

function buildConsentText(input: SubmissionZipInput): string {
  return (
    input.consentMarkdown +
    `\n--- SIGNATURE ---\n` +
    `Signed by: ${input.submission.consentSignatureName}\n` +
    `Accepted at: ${input.submission.consentAcceptedAt.toISOString()}\n` +
    `Document hash (sha256): ${input.submission.consentHash}\n`
  );
}

function slug(s: string): string {
  return s.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, "") || "unknown";
}

const EXPORT_NOTES = `Cells beginning with =, +, -, @, tab, or carriage-return have been prefixed with a single quote (')
to prevent spreadsheet formula evaluation when these CSVs are opened in Excel/Sheets/etc. Strip
the leading single quote to recover the original cell content. The original bytes remain in the
submission database untouched.\n`;
```

- [ ] **Step 5: Run, expect pass**

```bash
pnpm test tests/unit/zip-perSubmission.test.ts
```
Expected: PASS (1).

- [ ] **Step 6: Commit**

```bash
git add lib/zip/perSubmission.ts tests/unit/zip-perSubmission.test.ts package.json pnpm-lock.yaml
git commit -m "feat(zip): per-submission zip with sanitized CSVs + signed consent block"
```

---

### Task 26: Per-submission download route + admin detail page

**Files:**
- Create: `app/(admin)/admin/[id]/page.tsx`
- Create: `app/(admin)/admin/[id]/download/route.ts`
- Create: `lib/submissions/loadDetail.ts`

- [ ] **Step 1: Create lib/submissions/loadDetail.ts**

```ts
import { eq, desc } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { auditLog, csvFiles, submissions } from "@/lib/db/schema";

export async function loadSubmissionDetail(id: string) {
  const [row] = await db.select().from(submissions).where(eq(submissions.id, id));
  if (!row) return null;
  const files = await db.select().from(csvFiles).where(eq(csvFiles.submissionId, id));
  const audit = await db
    .select()
    .from(auditLog)
    .where(eq(auditLog.targetId, id))
    .orderBy(desc(auditLog.at));
  return { submission: row, files, audit };
}
```

- [ ] **Step 2: Create app/(admin)/admin/[id]/page.tsx**

```tsx
import { notFound } from "next/navigation";
import Link from "next/link";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { loadSubmissionDetail } from "@/lib/submissions/loadDetail";
import { logAudit } from "@/lib/audit/log";

export default async function SubmissionDetailPage({ params }: { params: Promise<{ id: string }> }) {
  const { email } = await requireAdmin();
  const { id } = await params;
  const data = await loadSubmissionDetail(id);
  if (!data) notFound();
  await logAudit({ event: "admin.viewed", actorType: "admin", actorId: email, targetId: id });

  const { submission, files, audit } = data;
  const videos = files.find((f) => f.kind === "videos");
  const followers = files.find((f) => f.kind === "followers");
  const status = submission.withdrawnAt ? "withdrawn" : submission.supersededAt ? "superseded" : "current";

  return (
    <main className="mx-auto max-w-3xl p-8">
      <Link href="/admin" className="text-sm underline">
        ← All submissions
      </Link>
      <h1 className="mt-4 text-2xl font-semibold">Submission {submission.id.slice(0, 8)}</h1>
      <p className="text-sm text-neutral-500">Status: {status}</p>

      <section className="mt-6">
        <h2 className="text-lg font-semibold">Metadata</h2>
        <dl className="mt-2 grid grid-cols-[max-content_1fr] gap-x-4 gap-y-1 text-sm">
          <dt className="text-neutral-500">Handle</dt>
          <dd>{submission.handleAtSubmission}</dd>
          <dt className="text-neutral-500">Email</dt>
          <dd>{submission.contactEmail}</dd>
          <dt className="text-neutral-500">Submitted</dt>
          <dd>{submission.submittedAt.toISOString()}</dd>
        </dl>
      </section>

      <section className="mt-6">
        <h2 className="text-lg font-semibold">Consent</h2>
        <dl className="mt-2 grid grid-cols-[max-content_1fr] gap-x-4 gap-y-1 text-sm">
          <dt className="text-neutral-500">Version</dt>
          <dd>{submission.consentVersion}</dd>
          <dt className="text-neutral-500">Hash</dt>
          <dd className="font-mono">{submission.consentHash}</dd>
          <dt className="text-neutral-500">Signature</dt>
          <dd>{submission.consentSignatureName}</dd>
          <dt className="text-neutral-500">Accepted</dt>
          <dd>{submission.consentAcceptedAt.toISOString()}</dd>
        </dl>
      </section>

      <section className="mt-6">
        <h2 className="text-lg font-semibold">Files</h2>
        <ul className="mt-2 space-y-2 text-sm">
          {[videos, followers].filter(Boolean).map((f) => (
            <li key={f!.id} className="rounded border p-3">
              <div className="font-mono">{f!.originalFilename}</div>
              <div className="text-neutral-500">
                {f!.kind} · {f!.sizeBytes} bytes · {f!.rowCount} rows
              </div>
              <div className="font-mono text-xs text-neutral-500">sha256 {f!.sha256}</div>
            </li>
          ))}
        </ul>
        <Link
          href={`/admin/${submission.id}/download`}
          className="mt-4 inline-block rounded bg-black px-3 py-1.5 text-sm text-white"
        >
          Download zip
        </Link>
      </section>

      <section className="mt-6">
        <h2 className="text-lg font-semibold">Actions</h2>
        <form
          action={async () => {
            "use server";
            const { withdrawAction } = await import("@/app/actions/withdraw");
            await withdrawAction(submission.id);
          }}
          className="mt-2"
        >
          <button className="rounded border border-red-600 px-3 py-1.5 text-sm text-red-700">
            Withdraw
          </button>
        </form>
        {submission.withdrawnAt ? (
          <form
            action={async () => {
              "use server";
              const { purgeAction } = await import("@/app/actions/purge");
              await purgeAction(submission.id);
            }}
            className="mt-2"
          >
            <button className="rounded bg-red-700 px-3 py-1.5 text-sm text-white">
              Purge permanently
            </button>
          </form>
        ) : null}
      </section>

      <section className="mt-6">
        <h2 className="text-lg font-semibold">Audit</h2>
        <ul className="mt-2 space-y-1 text-xs font-mono">
          {audit.map((a) => (
            <li key={a.id}>
              {a.at.toISOString()} · {a.actorType} · {a.actorId} · {a.event}
            </li>
          ))}
        </ul>
      </section>
    </main>
  );
}
```

- [ ] **Step 3: Create app/(admin)/admin/[id]/download/route.ts**

```ts
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { auditLog, csvFiles, submissions } from "@/lib/db/schema";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { buildSubmissionZip } from "@/lib/zip/perSubmission";
import { readCurrentConsent } from "@/lib/consent";
import { logAudit } from "@/lib/audit/log";

export async function GET(_req: Request, ctx: { params: Promise<{ id: string }> }) {
  const { email } = await requireAdmin();
  const { id } = await ctx.params;

  const [row] = await db.select().from(submissions).where(eq(submissions.id, id));
  if (!row) return new Response("not found", { status: 404 });
  const files = await db.select().from(csvFiles).where(eq(csvFiles.submissionId, id));
  const videos = files.find((f) => f.kind === "videos");
  const followers = files.find((f) => f.kind === "followers");
  if (!videos?.bytes || !followers?.bytes) {
    return new Response("submission files were withdrawn or superseded", { status: 410 });
  }
  const audit = await db.select().from(auditLog).where(eq(auditLog.targetId, id));

  await logAudit({
    event: "admin.downloaded",
    actorType: "admin",
    actorId: email,
    targetId: id,
  });

  const consent = readCurrentConsent();
  const stream = buildSubmissionZip({
    submission: row,
    videosFile: { originalFilename: videos.originalFilename, bytes: videos.bytes },
    followersFile: { originalFilename: followers.originalFilename, bytes: followers.bytes },
    consentMarkdown: consent.markdown,
    auditEntries: audit.map((a) => ({
      at: a.at.toISOString(),
      actorType: a.actorType,
      actorId: a.actorId,
      event: a.event,
      metadata: a.metadata,
    })),
  });

  return new Response(stream as unknown as ReadableStream, {
    headers: {
      "Content-Type": "application/zip",
      "Content-Disposition": `attachment; filename="submission_${id.slice(0, 8)}.zip"`,
    },
  });
}
```

- [ ] **Step 4: Commit**

```bash
git add app/\(admin\)/admin/\[id\]/ lib/submissions/loadDetail.ts
git commit -m "feat(admin): submission detail page + per-submission zip download"
```

---

### Task 27: Withdraw + purge server actions

**Files:**
- Create: `app/actions/withdraw.ts`
- Create: `app/actions/purge.ts`

- [ ] **Step 1: Create app/actions/withdraw.ts**

```ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { csvFiles, submissions } from "@/lib/db/schema";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { logAudit } from "@/lib/audit/log";

export async function withdrawAction(id: string, reason: string = "(no reason given)"): Promise<void> {
  const { email } = await requireAdmin();
  await db.transaction(async (tx) => {
    const [row] = await tx.select().from(submissions).where(eq(submissions.id, id));
    if (!row) return;
    await tx
      .update(submissions)
      .set({ withdrawnAt: new Date(), withdrawnBy: email, withdrawnReason: reason })
      .where(eq(submissions.id, id));
    await tx.update(csvFiles).set({ bytes: null }).where(eq(csvFiles.submissionId, id));
    await logAudit({
      event: "submission.withdrawn",
      actorType: "admin",
      actorId: email,
      targetId: id,
      metadata: { reason },
    });
  });
  revalidatePath(`/admin/${id}`);
  redirect(`/admin/${id}`);
}
```

- [ ] **Step 2: Create app/actions/purge.ts**

```ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { and, eq, isNotNull } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { submissions } from "@/lib/db/schema";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { logAudit } from "@/lib/audit/log";

export async function purgeAction(id: string): Promise<void> {
  const { email } = await requireAdmin();
  const result = await db.transaction(async (tx) => {
    const [row] = await tx
      .select()
      .from(submissions)
      .where(and(eq(submissions.id, id), isNotNull(submissions.withdrawnAt)));
    if (!row) return { ok: false as const };
    await tx.delete(submissions).where(eq(submissions.id, id));
    await logAudit({
      event: "submission.purged",
      actorType: "admin",
      actorId: email,
      targetId: id,
      metadata: {
        handle: row.handleAtSubmission,
        email: row.contactEmail,
        submitted_at: row.submittedAt.toISOString(),
      },
    });
    return { ok: true as const };
  });
  revalidatePath("/admin");
  if (result.ok) redirect("/admin");
  else redirect(`/admin/${id}`);
}
```

- [ ] **Step 3: Commit**

```bash
git add app/actions/withdraw.ts app/actions/purge.ts
git commit -m "feat(admin): withdraw + purge server actions with audit"
```

---

### Task 28: Bulk download all current submissions

**Files:**
- Create: `lib/zip/bulk.ts`
- Create: `app/(admin)/admin/download-all/route.ts`

- [ ] **Step 1: Create lib/zip/bulk.ts**

```ts
import archiver from "archiver";
import { Readable } from "node:stream";
import { and, eq, isNull } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { csvFiles, invites, submissions } from "@/lib/db/schema";
import { sanitizeCsvForExport } from "@/lib/csv/defensive";
import { readCurrentConsent } from "@/lib/consent";
import { stringify } from "csv-stringify/sync";

export async function buildBulkZip(): Promise<Readable> {
  const rows = await db
    .select({ submission: submissions, invite: invites })
    .from(submissions)
    .innerJoin(invites, eq(submissions.inviteId, invites.id))
    .where(and(isNull(submissions.supersededAt), isNull(submissions.withdrawnAt)));

  const archive = archiver("zip", { zlib: { level: 9 } });
  const rosterRows: string[][] = [
    ["pseudonymous_id", "creator_handle", "email", "submission_id", "submitted_at", "consent_version"],
  ];
  const consent = readCurrentConsent();

  for (const { submission, invite } of rows) {
    const files = await db.select().from(csvFiles).where(eq(csvFiles.submissionId, submission.id));
    const videos = files.find((f) => f.kind === "videos");
    const followers = files.find((f) => f.kind === "followers");
    if (!videos?.bytes || !followers?.bytes) continue;
    const folder = `submission_${submission.id.slice(0, 8)}_${slug(submission.handleAtSubmission)}`;
    archive.append(
      JSON.stringify(
        {
          submission_id: submission.id,
          handle: submission.handleAtSubmission,
          contact_email: submission.contactEmail,
          consent_version: submission.consentVersion,
          consent_hash: submission.consentHash,
          consent_signature_name: submission.consentSignatureName,
          consent_accepted_at: submission.consentAcceptedAt.toISOString(),
          submitted_at: submission.submittedAt.toISOString(),
        },
        null,
        2,
      ),
      { name: `${folder}/submission.json` },
    );
    archive.append(sanitizeCsvForExport(videos.bytes), {
      name: `${folder}/${videos.originalFilename}`,
    });
    archive.append(sanitizeCsvForExport(followers.bytes), {
      name: `${folder}/${followers.originalFilename}`,
    });
    archive.append(
      consent.markdown +
        `\n--- SIGNATURE ---\nSigned by: ${submission.consentSignatureName}\nAccepted at: ${submission.consentAcceptedAt.toISOString()}\nDocument hash (sha256): ${submission.consentHash}\n`,
      { name: `${folder}/consent_${submission.consentVersion}.txt` },
    );
    rosterRows.push([
      submission.id,
      submission.handleAtSubmission,
      invite.email,
      submission.id,
      submission.submittedAt.toISOString().slice(0, 10),
      submission.consentVersion,
    ]);
  }

  archive.append(stringify(rosterRows), { name: "roster.csv" });
  archive.finalize();
  return archive;
}

function slug(s: string): string {
  return s.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, "") || "unknown";
}
```

- [ ] **Step 2: Create app/(admin)/admin/download-all/route.ts**

```ts
import { requireAdmin } from "@/lib/auth/adminGuard";
import { buildBulkZip } from "@/lib/zip/bulk";
import { logAudit } from "@/lib/audit/log";

export async function GET() {
  const { email } = await requireAdmin();
  await logAudit({ event: "admin.downloaded", actorType: "admin", actorId: email, targetId: "bulk" });
  const stream = await buildBulkZip();
  return new Response(stream as unknown as ReadableStream, {
    headers: {
      "Content-Type": "application/zip",
      "Content-Disposition": `attachment; filename="sfv-submissions-${new Date().toISOString().slice(0, 10)}.zip"`,
    },
  });
}
```

- [ ] **Step 3: Commit**

```bash
git add lib/zip/bulk.ts app/\(admin\)/admin/download-all/
git commit -m "feat(admin): bulk download all current submissions + roster.csv"
```

---

## Phase 8 — Admin invites

### Task 29: Invites list page

**Files:**
- Create: `app/(admin)/admin/invites/page.tsx`
- Create: `app/(admin)/admin/invites/_components/InvitesTable.tsx`
- Create: `app/(admin)/admin/invites/_components/IssueInviteForm.tsx`
- Create: `app/(admin)/admin/invites/_components/BulkImportForm.tsx`
- Create: `lib/invites/list.ts`

- [ ] **Step 1: Create lib/invites/list.ts**

```ts
import { desc, eq, sql } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { invites, submissions } from "@/lib/db/schema";
import { formatAccessCode } from "@/lib/code/generate";

export interface InviteRow {
  id: string;
  maskedCode: string;
  handle: string;
  email: string;
  createdAt: string;
  createdBy: string;
  status: "active" | "revoked";
  uses: number;
  lastUsedAt: string | null;
}

export async function listInvites(): Promise<InviteRow[]> {
  const rows = await db
    .select({
      id: invites.id,
      code: invites.accessCode,
      handle: invites.creatorHandle,
      email: invites.email,
      createdAt: invites.createdAt,
      createdBy: invites.createdBy,
      revokedAt: invites.revokedAt,
      lastUsedAt: invites.lastUsedAt,
      uses: sql<number>`(SELECT COUNT(*)::int FROM ${submissions} s WHERE s.invite_id = ${invites.id})`,
    })
    .from(invites)
    .orderBy(desc(invites.createdAt));

  return rows.map((r) => ({
    id: r.id,
    maskedCode: mask(formatAccessCode(r.code)),
    handle: r.handle,
    email: r.email,
    createdAt: r.createdAt.toISOString().slice(0, 10),
    createdBy: r.createdBy,
    status: r.revokedAt ? "revoked" : "active",
    uses: Number(r.uses ?? 0),
    lastUsedAt: r.lastUsedAt ? r.lastUsedAt.toISOString().slice(0, 10) : null,
  }));
}

function mask(formatted: string): string {
  return formatted.replace(/^[^-]+-[^-]+/, (m) => "•".repeat(m.length));
}
```

- [ ] **Step 2: Create app/(admin)/admin/invites/_components/InvitesTable.tsx**

```tsx
import type { InviteRow } from "@/lib/invites/list";

export function InvitesTable({ rows }: { rows: InviteRow[] }) {
  if (rows.length === 0) return <p className="mt-4 text-sm text-neutral-500">No invites yet.</p>;
  return (
    <table className="mt-4 w-full text-sm">
      <thead>
        <tr className="border-b text-left">
          <th className="py-2">Code</th>
          <th>Handle</th>
          <th>Email</th>
          <th>Status</th>
          <th>Uses</th>
          <th>Last used</th>
          <th>Created</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        {rows.map((r) => (
          <tr key={r.id} className="border-b">
            <td className="py-2 font-mono">{r.maskedCode}</td>
            <td>{r.handle}</td>
            <td>{r.email}</td>
            <td>{r.status}</td>
            <td>{r.uses}</td>
            <td>{r.lastUsedAt ?? "—"}</td>
            <td>{r.createdAt}</td>
            <td className="space-x-2">
              <form
                action={async () => {
                  "use server";
                  const { revealCodeAction } = await import("@/app/actions/invites");
                  await revealCodeAction(r.id);
                }}
                className="inline"
              >
                <button className="text-xs underline">Reveal</button>
              </form>
              {r.status === "active" ? (
                <>
                  <form
                    action={async () => {
                      "use server";
                      const { revokeInviteAction } = await import("@/app/actions/invites");
                      await revokeInviteAction(r.id);
                    }}
                    className="inline"
                  >
                    <button className="text-xs underline">Revoke</button>
                  </form>
                  <form
                    action={async () => {
                      "use server";
                      const { reissueInviteAction } = await import("@/app/actions/invites");
                      await reissueInviteAction(r.id);
                    }}
                    className="inline"
                  >
                    <button className="text-xs underline">Reissue</button>
                  </form>
                </>
              ) : null}
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 3: Create app/(admin)/admin/invites/_components/IssueInviteForm.tsx**

```tsx
"use client";

import { useActionState } from "react";
import { issueInviteAction, type IssueInviteState } from "@/app/actions/invites";

const initial: IssueInviteState = { error: null, lastIssuedCode: null };

export function IssueInviteForm() {
  const [state, action, pending] = useActionState(issueInviteAction, initial);
  return (
    <form action={action} className="mt-4 grid grid-cols-[1fr_1fr_max-content] gap-2">
      <input name="handle" placeholder="TikTok handle" required className="rounded border px-3 py-2 text-sm" />
      <input name="email" placeholder="Email" type="email" required className="rounded border px-3 py-2 text-sm" />
      <button type="submit" disabled={pending} className="rounded bg-black px-3 py-2 text-sm text-white disabled:opacity-50">
        {pending ? "Sending..." : "Issue + email"}
      </button>
      {state.error ? <p className="col-span-3 text-sm text-red-600">{state.error}</p> : null}
      {state.lastIssuedCode ? (
        <p className="col-span-3 text-sm">
          Code issued: <span className="font-mono">{state.lastIssuedCode}</span>
        </p>
      ) : null}
    </form>
  );
}
```

- [ ] **Step 4: Create app/(admin)/admin/invites/_components/BulkImportForm.tsx**

```tsx
"use client";

import { useActionState } from "react";
import { bulkImportAction, type BulkImportState } from "@/app/actions/invites";

const initial: BulkImportState = { error: null, results: [] };

export function BulkImportForm() {
  const [state, action, pending] = useActionState<BulkImportState, FormData>(bulkImportAction, initial);
  return (
    <form action={action} encType="multipart/form-data" className="mt-4 space-y-2">
      <label className="block text-sm">Bulk import CSV (columns: handle, email)</label>
      <input name="file" type="file" accept=".csv" required className="block text-sm" />
      <button type="submit" disabled={pending} className="rounded bg-black px-3 py-2 text-sm text-white disabled:opacity-50">
        {pending ? "Processing..." : "Import + email"}
      </button>
      {state.error ? <p className="text-sm text-red-600">{state.error}</p> : null}
      {state.results.length > 0 ? (
        <ul className="text-sm">
          {state.results.map((r, i) => (
            <li key={i}>
              {r.email}: {r.ok ? "ok" : `failed (${r.error})`}
            </li>
          ))}
        </ul>
      ) : null}
    </form>
  );
}
```

- [ ] **Step 5: Create app/(admin)/admin/invites/page.tsx**

```tsx
import Link from "next/link";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { listInvites } from "@/lib/invites/list";
import { InvitesTable } from "./_components/InvitesTable";
import { IssueInviteForm } from "./_components/IssueInviteForm";
import { BulkImportForm } from "./_components/BulkImportForm";

export default async function InvitesPage() {
  await requireAdmin();
  const rows = await listInvites();
  return (
    <main className="mx-auto max-w-5xl p-8">
      <Link href="/admin" className="text-sm underline">
        ← All submissions
      </Link>
      <h1 className="mt-4 text-2xl font-semibold">Invites</h1>

      <section className="mt-6">
        <h2 className="text-lg font-semibold">Issue one</h2>
        <IssueInviteForm />
      </section>

      <section className="mt-8">
        <h2 className="text-lg font-semibold">Bulk import</h2>
        <BulkImportForm />
      </section>

      <section className="mt-8">
        <h2 className="text-lg font-semibold">All invites</h2>
        <InvitesTable rows={rows} />
      </section>
    </main>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add app/\(admin\)/admin/invites/page.tsx app/\(admin\)/admin/invites/_components/ lib/invites/
git commit -m "feat(admin): invites list page + issue + bulk import forms"
```

---

### Task 30: Invite server actions (issue / bulk / revoke / reissue / reveal)

**Files:**
- Create: `app/actions/invites.ts`
- Create: `lib/invites/create.ts`

- [ ] **Step 1: Create lib/invites/create.ts**

```ts
import { z } from "zod";
import { generateAccessCode, formatAccessCode } from "@/lib/code/generate";
import { db } from "@/lib/db/client";
import { invites } from "@/lib/db/schema";
import { getMailTransport } from "@/lib/mail/transport";
import { inviteEmail } from "@/lib/mail/templates/invite";
import { env } from "@/lib/env";
import { logAudit } from "@/lib/audit/log";

const schema = z.object({
  handle: z.string().trim().min(1),
  email: z.string().trim().email(),
});

export type IssueResult =
  | { ok: true; id: string; formattedCode: string }
  | { ok: false; error: string };

export async function issueInvite(args: {
  handle: string;
  email: string;
  createdBy: string;
}): Promise<IssueResult> {
  const parsed = schema.safeParse({ handle: args.handle, email: args.email });
  if (!parsed.success) return { ok: false, error: parsed.error.issues[0]?.message ?? "Invalid input." };

  const code = generateAccessCode();
  const [row] = await db
    .insert(invites)
    .values({
      accessCode: code,
      creatorHandle: parsed.data.handle,
      email: parsed.data.email,
      createdBy: args.createdBy,
    })
    .returning({ id: invites.id });

  await logAudit({
    event: "invite.issued",
    actorType: "admin",
    actorId: args.createdBy,
    targetId: row.id,
    metadata: { handle: parsed.data.handle, email: parsed.data.email },
  });

  const message = inviteEmail({
    handle: parsed.data.handle,
    code: formatAccessCode(code),
    siteUrl: env.SITE_URL,
    contactEmail: env.RESEARCH_CONTACT_EMAIL,
  });
  const sent = await getMailTransport().send({ to: parsed.data.email, ...message });
  if (!sent.ok) {
    await logAudit({
      event: "invite.email_failed",
      actorType: "admin",
      actorId: args.createdBy,
      targetId: row.id,
      metadata: { error: sent.error ?? "unknown" },
    });
  }
  return { ok: true, id: row.id, formattedCode: formatAccessCode(code) };
}
```

- [ ] **Step 2: Create app/actions/invites.ts**

```ts
"use server";

import { parse } from "csv-parse/sync";
import { revalidatePath } from "next/cache";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db/client";
import { invites } from "@/lib/db/schema";
import { requireAdmin } from "@/lib/auth/adminGuard";
import { issueInvite } from "@/lib/invites/create";
import { formatAccessCode, generateAccessCode } from "@/lib/code/generate";
import { getMailTransport } from "@/lib/mail/transport";
import { reissueEmail } from "@/lib/mail/templates/reissue";
import { env } from "@/lib/env";
import { logAudit } from "@/lib/audit/log";

export interface IssueInviteState {
  error: string | null;
  lastIssuedCode: string | null;
}

export async function issueInviteAction(_prev: IssueInviteState, formData: FormData): Promise<IssueInviteState> {
  const { email: adminEmail } = await requireAdmin();
  const handle = String(formData.get("handle") ?? "");
  const email = String(formData.get("email") ?? "");
  const r = await issueInvite({ handle, email, createdBy: adminEmail });
  revalidatePath("/admin/invites");
  if (!r.ok) return { error: r.error, lastIssuedCode: null };
  return { error: null, lastIssuedCode: r.formattedCode };
}

export interface BulkImportState {
  error: string | null;
  results: { email: string; ok: boolean; error?: string }[];
}

export async function bulkImportAction(_prev: BulkImportState, formData: FormData): Promise<BulkImportState> {
  const { email: adminEmail } = await requireAdmin();
  const file = formData.get("file") as File | null;
  if (!file || file.size === 0) return { error: "Choose a CSV file.", results: [] };
  const bytes = Buffer.from(await file.arrayBuffer());
  let rows: string[][];
  try {
    rows = parse(bytes, { bom: true, columns: false }) as string[][];
  } catch (e) {
    return { error: `CSV parse failed: ${(e as Error).message}`, results: [] };
  }
  if (rows.length === 0) return { error: "CSV is empty.", results: [] };
  const [header, ...data] = rows;
  const handleIdx = header.findIndex((h) => h.trim().toLowerCase() === "handle");
  const emailIdx = header.findIndex((h) => h.trim().toLowerCase() === "email");
  if (handleIdx < 0 || emailIdx < 0) {
    return { error: 'CSV must include "handle" and "email" columns.', results: [] };
  }
  const results: BulkImportState["results"] = [];
  for (const row of data) {
    const handle = row[handleIdx];
    const email = row[emailIdx];
    if (!handle || !email) {
      results.push({ email: email ?? "(missing)", ok: false, error: "missing field" });
      continue;
    }
    const r = await issueInvite({ handle, email, createdBy: adminEmail });
    if (r.ok) results.push({ email, ok: true });
    else results.push({ email, ok: false, error: r.error });
  }
  revalidatePath("/admin/invites");
  return { error: null, results };
}

export async function revokeInviteAction(id: string): Promise<void> {
  const { email: adminEmail } = await requireAdmin();
  await db.update(invites).set({ revokedAt: new Date() }).where(eq(invites.id, id));
  await logAudit({ event: "invite.revoked", actorType: "admin", actorId: adminEmail, targetId: id });
  revalidatePath("/admin/invites");
}

export async function reissueInviteAction(id: string): Promise<void> {
  const { email: adminEmail } = await requireAdmin();
  const [old] = await db.select().from(invites).where(eq(invites.id, id));
  if (!old) return;
  await db.update(invites).set({ revokedAt: new Date() }).where(eq(invites.id, id));
  await logAudit({ event: "invite.revoked", actorType: "admin", actorId: adminEmail, targetId: id });

  const code = generateAccessCode();
  const [created] = await db
    .insert(invites)
    .values({ accessCode: code, creatorHandle: old.creatorHandle, email: old.email, createdBy: adminEmail })
    .returning({ id: invites.id });
  await logAudit({
    event: "invite.issued",
    actorType: "admin",
    actorId: adminEmail,
    targetId: created.id,
    metadata: { reissued_from: id },
  });
  const message = reissueEmail({
    handle: old.creatorHandle,
    code: formatAccessCode(code),
    siteUrl: env.SITE_URL,
    contactEmail: env.RESEARCH_CONTACT_EMAIL,
  });
  await getMailTransport().send({ to: old.email, ...message });
  revalidatePath("/admin/invites");
}

export async function revealCodeAction(id: string): Promise<void> {
  const { email: adminEmail } = await requireAdmin();
  const [row] = await db.select().from(invites).where(eq(invites.id, id));
  if (!row) return;
  console.log(`[admin reveal by ${adminEmail}] invite ${id} code=${formatAccessCode(row.accessCode)}`);
  await logAudit({
    event: "admin.viewed",
    actorType: "admin",
    actorId: adminEmail,
    targetId: id,
    metadata: { action: "reveal_code" },
  });
}
```

- [ ] **Step 3: End-to-end check (manual)**

Sign into /admin/login, navigate to /admin/invites, issue an invite to your own email. Confirm:
- Row appears in the table
- For dev-log transport: code prints to the `pnpm dev` console
- For SMTP: email arrives

- [ ] **Step 4: Commit**

```bash
git add app/actions/invites.ts lib/invites/create.ts
git commit -m "feat(admin): invite server actions (issue, bulk, revoke, reissue, reveal)"
```

---

## Phase 9 — Integration tests + smoke

### Task 31: Integration test harness with testcontainers

**Files:**
- Create: `tests/integration/_helpers/db.ts`
- Create: `tests/integration/enterCode.test.ts`

- [ ] **Step 1: Create tests/integration/_helpers/db.ts**

```ts
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { drizzle } from "drizzle-orm/node-postgres";
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { Pool } from "pg";
import * as schema from "@/lib/db/schema";

export interface TestDb {
  container: StartedPostgreSqlContainer;
  pool: Pool;
  db: ReturnType<typeof drizzle<typeof schema>>;
  url: string;
  stop: () => Promise<void>;
}

export async function startTestDb(): Promise<TestDb> {
  const container = await new PostgreSqlContainer("postgres:16").start();
  const url = container.getConnectionUri();
  process.env.DATABASE_URL = url;
  const pool = new Pool({ connectionString: url });
  const db = drizzle(pool, { schema });
  await migrate(db, { migrationsFolder: "./drizzle" });
  return {
    container,
    pool,
    db,
    url,
    stop: async () => {
      await pool.end();
      await container.stop();
    },
  };
}
```

- [ ] **Step 2: Write integration test for enterCode**

```ts
import { afterAll, beforeAll, beforeEach, describe, expect, it } from "vitest";
import { startTestDb, type TestDb } from "./_helpers/db";
import { invites, auditLog } from "@/lib/db/schema";
import { eq } from "drizzle-orm";

let t: TestDb;
beforeAll(async () => {
  t = await startTestDb();
  process.env.PARTICIPANT_SESSION_SECRET = "a".repeat(40);
});
afterAll(async () => t?.stop());
beforeEach(async () => {
  await t.db.delete(auditLog);
  await t.db.delete(invites);
});

describe("enterCode (integration)", () => {
  it("creates a code.entered audit row for a valid code", async () => {
    const code = "ABCD2345MNPQ";
    await t.db.insert(invites).values({
      accessCode: code,
      creatorHandle: "aki",
      email: "aki@example.com",
      createdBy: "test",
    });
    const { verifyAccessCode } = await import("@/lib/code/verify");
    const r = await verifyAccessCode("abcd-2345-mnpq");
    expect(r.ok).toBe(true);
    if (r.ok) {
      expect(r.invite.creatorHandle).toBe("aki");
    }
  });

  it("returns revoked for revoked code", async () => {
    const code = "REVK2345MNPQ";
    await t.db.insert(invites).values({
      accessCode: code,
      creatorHandle: "x",
      email: "x@x.com",
      createdBy: "test",
      revokedAt: new Date(),
    });
    const { verifyAccessCode } = await import("@/lib/code/verify");
    const r = await verifyAccessCode(code);
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.reason).toBe("revoked");
  });
});
```

- [ ] **Step 3: Run integration tests**

```bash
pnpm test:integration
```
Expected: PASS (2). Docker must be running.

- [ ] **Step 4: Commit**

```bash
git add tests/integration/
git commit -m "test(integration): testcontainer Postgres harness + enterCode coverage"
```

---

### Task 32: Integration test for createSubmission (supersede flow)

**Files:**
- Create: `tests/integration/createSubmission.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { afterAll, beforeAll, beforeEach, describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { eq, isNotNull } from "drizzle-orm";
import { startTestDb, type TestDb } from "./_helpers/db";
import { invites, submissions, csvFiles, auditLog } from "@/lib/db/schema";
import { createSubmission } from "@/lib/submissions/create";
import { validateCsv } from "@/lib/csv/validate";

const videosBytes = readFileSync("tests/fixtures/tiktok_videos_unknown_2026-06-30.csv");
const followersBytes = readFileSync("tests/fixtures/tiktok_followers_unknown_2026-06-30.csv");

let t: TestDb;
beforeAll(async () => {
  t = await startTestDb();
  process.env.PARTICIPANT_SESSION_SECRET = "a".repeat(40);
});
afterAll(async () => t?.stop());
beforeEach(async () => {
  await t.db.delete(auditLog);
  await t.db.delete(csvFiles);
  await t.db.delete(submissions);
  await t.db.delete(invites);
});

async function seedInvite() {
  const [row] = await t.db
    .insert(invites)
    .values({
      accessCode: "TESTCODE1234",
      creatorHandle: "aki",
      email: "aki@example.com",
      createdBy: "seed",
    })
    .returning({ id: invites.id });
  return row.id;
}

function validate(name: string, bytes: Buffer) {
  const r = validateCsv({ name, bytes });
  if (!r.ok) throw new Error("fixture should validate");
  return r;
}

describe("createSubmission", () => {
  it("inserts a new submission with both csv files and audit rows", async () => {
    const inviteId = await seedInvite();
    const id = await createSubmission(t.db, {
      inviteId,
      handle: "aki",
      contactEmail: "aki@example.com",
      consent: { version: "v1.0", hash: "h", signatureName: "Aki", acceptedAt: Date.now() },
      videos: validate("tiktok_videos_aki_2026-06-30.csv", videosBytes),
      followers: validate("tiktok_followers_aki_2026-06-30.csv", followersBytes),
    });
    expect(id).toMatch(/^[0-9a-f-]{36}$/);
    const files = await t.db.select().from(csvFiles).where(eq(csvFiles.submissionId, id));
    expect(files).toHaveLength(2);
    expect(files.find((f) => f.kind === "videos")?.bytes?.byteLength).toBe(videosBytes.byteLength);

    const events = await t.db.select({ event: auditLog.event }).from(auditLog);
    const codes = events.map((e) => e.event);
    expect(codes).toContain("submission.created");
    expect(codes.filter((c) => c === "csv.validated")).toHaveLength(2);
  });

  it("supersedes prior submission and nulls its bytes", async () => {
    const inviteId = await seedInvite();
    const baseConsent = { version: "v1.0", hash: "h", signatureName: "Aki", acceptedAt: Date.now() };
    const first = await createSubmission(t.db, {
      inviteId,
      handle: "aki",
      contactEmail: "aki@example.com",
      consent: baseConsent,
      videos: validate("tiktok_videos_aki_2026-06-30.csv", videosBytes),
      followers: validate("tiktok_followers_aki_2026-06-30.csv", followersBytes),
    });
    const second = await createSubmission(t.db, {
      inviteId,
      handle: "aki",
      contactEmail: "aki@example.com",
      consent: baseConsent,
      videos: validate("tiktok_videos_aki_2026-06-30.csv", videosBytes),
      followers: validate("tiktok_followers_aki_2026-06-30.csv", followersBytes),
    });
    expect(first).not.toBe(second);

    const supersededRows = await t.db
      .select()
      .from(submissions)
      .where(isNotNull(submissions.supersededAt));
    expect(supersededRows.map((r) => r.id)).toContain(first);

    const firstFiles = await t.db.select().from(csvFiles).where(eq(csvFiles.submissionId, first));
    expect(firstFiles.every((f) => f.bytes === null)).toBe(true);

    const secondFiles = await t.db.select().from(csvFiles).where(eq(csvFiles.submissionId, second));
    expect(secondFiles.every((f) => f.bytes !== null)).toBe(true);
  });
});
```

- [ ] **Step 2: Run, expect pass**

```bash
pnpm test:integration tests/integration/createSubmission.test.ts
```
Expected: PASS (2).

- [ ] **Step 3: Commit**

```bash
git add tests/integration/createSubmission.test.ts
git commit -m "test(integration): submission insert + supersede + bytes-null behavior"
```

---

### Task 33: Playwright smoke test

**Files:**
- Create: `playwright.config.ts`
- Create: `tests/smoke/happy-path.spec.ts`
- Create: `tests/smoke/_helpers/seed.ts`

- [ ] **Step 1: Initialize Playwright browsers**

```bash
pnpm playwright install chromium
```

- [ ] **Step 2: Create playwright.config.ts**

```ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/smoke",
  timeout: 60_000,
  use: {
    baseURL: "http://localhost:3000",
    headless: true,
    screenshot: "only-on-failure",
  },
  webServer: {
    command: "pnpm build && pnpm start",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

- [ ] **Step 3: Create tests/smoke/_helpers/seed.ts**

```ts
import { Pool } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "@/lib/db/schema";

export async function seedSmokeInvite() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL! });
  const db = drizzle(pool, { schema });
  const code = "SMOK2345MNPQ";
  await db
    .insert(schema.invites)
    .values({
      accessCode: code,
      creatorHandle: "smoke",
      email: "smoke@example.com",
      createdBy: "smoke-seed",
    })
    .onConflictDoNothing();
  await pool.end();
  return code;
}
```

- [ ] **Step 4: Write tests/smoke/happy-path.spec.ts**

```ts
import { test, expect } from "@playwright/test";
import { seedSmokeInvite } from "./_helpers/seed";

test("participant code → consent → submit → confirmation", async ({ page }) => {
  const code = await seedSmokeInvite();
  await page.goto("/");
  await page.fill("input[name=code]", code);
  await page.click("button:has-text('Continue')");

  await page.waitForURL("**/consent");
  await page.fill("input[name=signatureName]", "Smoke Tester");
  await page.check("input[name=agreed]");
  await page.click("button:has-text('Continue')");

  await page.waitForURL("**/submit");
  await page.setInputFiles("input[name=videos]", "tests/fixtures/tiktok_videos_unknown_2026-06-30.csv");
  await page.setInputFiles("input[name=followers]", "tests/fixtures/tiktok_followers_unknown_2026-06-30.csv");
  await page.click("button:has-text('Submit')");

  await page.waitForURL("**/confirmed/**");
  await expect(page.getByText("Submission received")).toBeVisible();
  await expect(page.getByText("Smoke Tester")).toBeVisible();
});
```

- [ ] **Step 5: Run smoke**

Ensure dev Postgres is running and `pnpm db:migrate` has been run against it. Then:
```bash
pnpm test:smoke
```
Expected: 1 passed.

- [ ] **Step 6: Commit**

```bash
git add playwright.config.ts tests/smoke/
git commit -m "test(smoke): happy-path participant flow with Playwright"
```

---

## Phase 10 — Deploy artifacts

### Task 34: nginx + systemd + deploy README

**Files:**
- Create: `deploy/nginx.conf.example`
- Create: `deploy/systemd/sfv-site.service.example`
- Create: `deploy/README.md`

- [ ] **Step 1: Create deploy/nginx.conf.example**

```nginx
server {
  listen 443 ssl http2;
  server_name submit.altdsi.example;

  ssl_certificate     /etc/letsencrypt/live/submit.altdsi.example/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/submit.altdsi.example/privkey.pem;

  client_max_body_size 10m;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 120s;
  }
}

server {
  listen 80;
  server_name submit.altdsi.example;
  return 301 https://$host$request_uri;
}
```

- [ ] **Step 2: Create deploy/systemd/sfv-site.service.example**

```
[Unit]
Description=SFV TikTok submission site (Next.js)
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=sfv
WorkingDirectory=/srv/sfv-site
EnvironmentFile=/srv/sfv-site/.env
ExecStart=/usr/bin/pnpm start
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

- [ ] **Step 3: Create deploy/README.md**

```md
# Deploy notes

Target: a Linux VM on ALTDSI with root.

## One-time

1. Install Node 20+ via NodeSource or nvm; enable pnpm via corepack.
2. Install PostgreSQL 16. Create db and user:
   ```sql
   CREATE USER sfv WITH PASSWORD '...';
   CREATE DATABASE sfv OWNER sfv;
   ```
3. Create a Linux service user `sfv` and clone the repo to `/srv/sfv-site`.
4. Copy `.env.example` to `/srv/sfv-site/.env` and fill in:
   - `DATABASE_URL` — `postgresql://sfv:...@localhost/sfv`
   - `PARTICIPANT_SESSION_SECRET` (32+ chars random)
   - `NEXTAUTH_URL` — full HTTPS origin
   - `NEXTAUTH_SECRET` (32+ chars random)
   - `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` (from ALTDSI sysadmin)
   - `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM` (from ALTDSI sysadmin)
   - `ADMIN_ALLOWLIST` — comma-separated approved emails
   - `SITE_URL`, `RESEARCH_CONTACT_EMAIL`
5. Place nginx config from `nginx.conf.example` under `/etc/nginx/sites-available/sfv-site`, link to `sites-enabled/`, run `certbot` for the cert.
6. Install systemd unit from `systemd/sfv-site.service.example` to `/etc/systemd/system/sfv-site.service`. `systemctl daemon-reload && systemctl enable --now sfv-site`.

## Release

```bash
sudo -u sfv -i
cd /srv/sfv-site
git pull
pnpm install --frozen-lockfile
pnpm db:migrate
pnpm db:seed-allowlist
pnpm build
sudo systemctl restart sfv-site
```

## Backup

Nightly cron on a researcher-controlled machine:
```bash
ssh sfv@altdsi-host pg_dump -Fc sfv > sfv-$(date -u +%F).dump
```

## Coupled-change reminder

When the Chrome extension bumps its CSV columns, bump `EXPECTED_SCHEMA_VERSION` and the header arrays in `lib/csv/schemas.ts` together. The validator will reject submissions otherwise.
```

- [ ] **Step 4: Commit**

```bash
git add deploy/
git commit -m "docs(deploy): nginx + systemd unit + deploy README"
```

---

### Task 35: Wire up confirmation email + admin.denied logging

These two emissions were missed in earlier tasks. Adding them here keeps Tasks 20 and 22 clean.

**Files:**
- Modify: `app/actions/submitForm.ts`
- Modify: `middleware.ts`

- [ ] **Step 1: Send confirmation email after a successful submission**

In `app/actions/submitForm.ts`, just before the final `redirect(\`/confirmed/${submissionId}\`)`:

```ts
import { confirmationEmail } from "@/lib/mail/templates/confirmation";
import { getMailTransport } from "@/lib/mail/transport";
import { env } from "@/lib/env";
import { logAudit } from "@/lib/audit/log";

// ...after `const submissionId = await createSubmission(...)`:
const isoDate = new Date().toISOString().slice(0, 10);
const message = confirmationEmail({
  shortId: submissionId.slice(0, 8),
  videoFilename: `tiktok_videos_${parsed.data!.handle}_${isoDate}.csv`,
  followerFilename: `tiktok_followers_${parsed.data!.handle}_${isoDate}.csv`,
  siteUrl: env.SITE_URL,
  contactEmail: env.RESEARCH_CONTACT_EMAIL,
});
const sent = await getMailTransport().send({ to: parsed.data!.contactEmail, ...message });
if (!sent.ok) {
  await logAudit({
    event: "system.error",
    actorType: "system",
    actorId: "confirmation-email",
    targetId: submissionId,
    metadata: { error: sent.error ?? "unknown" },
  });
}
```

- [ ] **Step 2: Emit admin.denied from middleware**

In `middleware.ts`, replace the allowlist-failure branch:

```ts
if (!ok) {
  await db.insert(auditLog).values({
    event: "admin.denied",
    actorType: "admin",
    actorId: session.user.email,
    targetId: null,
    metadata: { path: req.nextUrl.pathname },
  });
  const url = new URL("/admin/forbidden", req.url);
  return NextResponse.redirect(url);
}
```

Add `auditLog` to the import from `@/lib/db/schema` at the top of the file.

- [ ] **Step 3: Smoke-check**

- Submit through the participant flow; confirm the dev-log mail transport prints the confirmation email body.
- Sign into /admin with an email *not* in `ADMIN_ALLOWLIST`; confirm the request lands on `/admin/forbidden` and a row appears in `audit_log` with `event = 'admin.denied'`.

- [ ] **Step 4: Commit**

```bash
git add app/actions/submitForm.ts middleware.ts
git commit -m "feat: send confirmation email + log admin.denied from middleware"
```

---

## Self-review checklist (run before declaring done)

Run yourself, not the engineer:

- [x] **Spec coverage:** Every requirement in the spec is implemented by some task above.
- [x] **Audit events:** Every event in `AUDIT_EVENTS` is emitted by at least one task (`admin.denied` and `system.error` added in Task 35).
- [x] **Type/name consistency:** `ParticipantSessionData`, `ValidateResult`, `IssueResult`, etc. are used with identical shapes everywhere they appear.
- [x] **Commits:** Each task ends with a commit step.
- [x] **Test cadence:** Each non-config task has a TDD cycle where it makes sense (pure-config tasks like Task 1 and Task 34 verify by build/runtime check instead).

---

## CI (nice-to-have, not gated)

Add later: `.github/workflows/ci.yml` running `pnpm install --frozen-lockfile && pnpm lint && pnpm typecheck && pnpm test && pnpm test:integration && pnpm test:smoke` with a Postgres service container. Not in this plan to keep scope tight.

---

End of plan. The engineer follows tasks 1-35 in order. Open `docs/superpowers/specs/2026-06-30-altdsi-tiktok-submission-site-design.md` alongside for the *why* behind any decision.



