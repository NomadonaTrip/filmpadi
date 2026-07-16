# FilmPadi Foundation Plan 1 of 3 — Platform Substrate

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A running Next.js 15 app with the complete database schema, vendor adapters, module-boundary enforcement, CI, and the FilmPadi base layout — the substrate Plans 2 (auth) and 3 (catalog/SEO/events/search) build on.

**Architecture:** Single Next.js App Router app with vertical feature modules under `src/modules/`; all vendor access behind adapter interfaces in `src/lib/adapters/`; Drizzle ORM over local Postgres 16; trust-invariant module boundaries enforced by ESLint from day one.

**Tech Stack:** Next.js 15 (App Router, TypeScript strict), React 19, Tailwind CSS v4, Drizzle ORM + postgres.js, Postgres 16 (local via apt), Vitest 3 + Testing Library, ESLint 9 flat config, pnpm, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-07-16-filmpadi-foundation-design.md`

## Global Constraints

- Repo root is `~/projects/filmpadi`. All commands run there unless stated.
- Node 20.x, pnpm 10. TypeScript `strict: true` everywhere.
- Path alias `@/*` → `src/*`.
- Databases: `filmpadi_dev` (dev), `filmpadi_test` (tests) — local Postgres 16, role `filmpadi`/`filmpadi`.
- Money is always `bigint` kobo. Phones are always E.164 `text`.
- Vendor SDKs/APIs are only touched inside `src/lib/adapters/`.
- Module boundaries (ESLint-enforced, CI-blocking): `src/app/**` never imports `@/db*`; `src/modules/trending/**` and `src/modules/feed/**` never import `@/modules/campaigns*`; only `src/modules/events/**` and `src/db/**` may import `@/db/schema/events`.
- First-load JS < 150KB gzipped on viewer routes (CI-enforced from this plan onward). No third-party scripts on viewer surfaces.
- TDD where there is logic to test; every task ends with a commit.

---

### Task 1: App scaffold

Manual scaffold (no `create-next-app` — the repo already has `docs/` and `.git`, and manual is deterministic).

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `postcss.config.mjs`, `.gitignore`, `.env.example`, `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/globals.css`

**Interfaces:**
- Produces: the `@/*` path alias, `pnpm dev|build|typecheck` scripts, Tailwind v4 pipeline. Every later task assumes these.

- [ ] **Step 1: Write `package.json`**

```json
{
  "name": "filmpadi",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "tsx src/db/migrate.ts",
    "check:bundle": "node scripts/check-bundle-size.mjs"
  },
  "dependencies": {
    "drizzle-orm": "^0.44.2",
    "next": "^15.3.4",
    "postgres": "^3.4.5",
    "react": "^19.1.0",
    "react-dom": "^19.1.0",
    "zod": "^3.25.67"
  },
  "devDependencies": {
    "@eslint/eslintrc": "^3.3.1",
    "@tailwindcss/postcss": "^4.1.10",
    "@testing-library/react": "^16.3.0",
    "@types/node": "^20.19.0",
    "@types/react": "^19.1.8",
    "@types/react-dom": "^19.1.6",
    "@vitejs/plugin-react": "^4.5.2",
    "dotenv": "^16.5.0",
    "drizzle-kit": "^0.31.1",
    "eslint": "^9.29.0",
    "eslint-config-next": "^15.3.4",
    "jsdom": "^26.1.0",
    "tailwindcss": "^4.1.10",
    "tsx": "^4.20.3",
    "typescript": "^5.8.3",
    "vite-tsconfig-paths": "^5.1.4",
    "vitest": "^3.2.4"
  }
}
```

- [ ] **Step 2: Write `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: Write `next.config.ts`**

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    // Cloudflare Images domains added when R2 is provisioned; local dev serves /uploads
    remotePatterns: [],
  },
};

export default nextConfig;
```

- [ ] **Step 4: Write `postcss.config.mjs`**

```js
export default {
  plugins: { "@tailwindcss/postcss": {} },
};
```

- [ ] **Step 5: Write `src/app/globals.css`** (design tokens land properly in Task 11; this is the minimum)

```css
@import "tailwindcss";
```

- [ ] **Step 6: Write `src/app/layout.tsx`**

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: { default: "FilmPadi", template: "%s — FilmPadi" },
  description: "The complete, current picture of Nollywood.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

- [ ] **Step 7: Write `src/app/page.tsx`**

```tsx
export default function HomePage() {
  return (
    <main>
      <h1>FilmPadi</h1>
    </main>
  );
}
```

- [ ] **Step 8: Write `.gitignore`**

```
node_modules/
.next/
out/
coverage/
.env.local
*.tsbuildinfo
next-env.d.ts
public/uploads/
```

- [ ] **Step 9: Write `.env.example`** (full v1 variable set from the brief §11 plus local additions)

```
DATABASE_URL=postgres://filmpadi:filmpadi@localhost:5432/filmpadi_dev
SMS_PROVIDER=dev            # dev | termii
STORAGE_PROVIDER=local      # local | r2
TERMII_API_KEY=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
PAYSTACK_SECRET_KEY=
PAYSTACK_PUBLIC_KEY=
R2_ACCOUNT_ID=
R2_ACCESS_KEY=
R2_SECRET=
R2_BUCKET=
R2_PUBLIC_BASE_URL=
YOUTUBE_API_KEY=
CRON_SECRET=
TRENDING_MATURITY_N=25
```

- [ ] **Step 10: Install and verify build**

Run: `pnpm install && pnpm build`
Expected: build succeeds; route `/` listed in output.

- [ ] **Step 11: Verify dev server**

Run: `pnpm dev &` then `curl -s http://localhost:3000 | grep FilmPadi`; kill the dev server after.
Expected: HTML containing `FilmPadi`.

- [ ] **Step 12: Commit**

```bash
git add -A && git commit -m "feat: scaffold Next.js 15 app with Tailwind v4 and strict TS"
```

---

### Task 2: Test infrastructure (Vitest)

**Files:**
- Create: `vitest.config.ts`, `tests/setup.ts`, `tests/unit/smoke.test.ts`

**Interfaces:**
- Produces: `pnpm test` runs `tests/**/*.test.{ts,tsx}`; `.env.test` (created in Task 5) is auto-loaded; `@/*` alias works in tests. Component tests opt into jsdom with a `// @vitest-environment jsdom` pragma.

- [ ] **Step 1: Write `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    include: ["tests/**/*.test.{ts,tsx}"],
    setupFiles: ["tests/setup.ts"],
    environment: "node",
  },
});
```

- [ ] **Step 2: Write `tests/setup.ts`**

```ts
import { config } from "dotenv";

// Test env (test DATABASE_URL etc.). Committed file, local-only credentials.
config({ path: ".env.test" });
```

- [ ] **Step 3: Write `tests/unit/smoke.test.ts`**

```ts
import { describe, expect, it } from "vitest";

describe("test infrastructure", () => {
  it("runs", () => {
    expect(1 + 1).toBe(2);
  });
});
```

- [ ] **Step 4: Run tests**

Run: `pnpm test`
Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "test: add Vitest infrastructure"
```

---

### Task 3: Module skeleton + ESLint boundary rules

The trust invariant from the brief §0 ("trending and organic-feed modules must not import campaign data") becomes lint config now, while the directories are empty, so it is already law when Week 5/7 code arrives.

**Files:**
- Create: `eslint.config.mjs`, `src/modules/{auth,catalog,events,search,seo,trending,feed,campaigns}/.gitkeep`, `src/db/.gitkeep`, `tests/unit/eslint-boundaries.test.ts`

**Interfaces:**
- Produces: the three boundary rules from Global Constraints, live in `pnpm lint` and unit-tested. Later tasks place code in these module directories.

- [ ] **Step 1: Create the directory skeleton**

```bash
mkdir -p src/modules/{auth,catalog,events,search,seo,trending,feed,campaigns} src/db
touch src/modules/{auth,catalog,events,search,seo,trending,feed,campaigns}/.gitkeep src/db/.gitkeep
```

- [ ] **Step 2: Write the failing boundary test — `tests/unit/eslint-boundaries.test.ts`**

```ts
import { ESLint } from "eslint";
import { describe, expect, it } from "vitest";

const eslint = new ESLint({ cwd: process.cwd() });

async function boundaryViolations(code: string, filePath: string) {
  const results = await eslint.lintText(code, { filePath });
  return (results[0]?.messages ?? []).filter((m) => m.ruleId === "no-restricted-imports");
}

describe("module boundary rules", () => {
  it("forbids app/ from importing the db layer", async () => {
    const v = await boundaryViolations(`import { db } from "@/db/client";`, "src/app/film/[slug]/page.tsx");
    expect(v.length).toBeGreaterThan(0);
  });

  it("allows modules to import the db layer", async () => {
    const v = await boundaryViolations(`import { db } from "@/db/client";`, "src/modules/catalog/queries.ts");
    expect(v).toHaveLength(0);
  });

  it("forbids trending from importing campaigns (trust invariant)", async () => {
    const v = await boundaryViolations(`import { x } from "@/modules/campaigns/serving";`, "src/modules/trending/recompute.ts");
    expect(v.length).toBeGreaterThan(0);
  });

  it("forbids feed from importing campaigns (trust invariant)", async () => {
    const v = await boundaryViolations(`import { x } from "@/modules/campaigns/serving";`, "src/modules/feed/build.ts");
    expect(v.length).toBeGreaterThan(0);
  });

  it("forbids other modules from importing the events table schema", async () => {
    const v = await boundaryViolations(`import { events } from "@/db/schema/events";`, "src/modules/catalog/queries.ts");
    expect(v.length).toBeGreaterThan(0);
  });

  it("allows modules/events to import the events table schema", async () => {
    const v = await boundaryViolations(`import { events } from "@/db/schema/events";`, "src/modules/events/log.ts");
    expect(v).toHaveLength(0);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm test tests/unit/eslint-boundaries.test.ts`
Expected: FAIL (no eslint config / no violations reported yet).

- [ ] **Step 4: Write `eslint.config.mjs`**

Rule-merge note: in flat config, a later object's `no-restricted-imports` **replaces** an earlier one for matching files — so each object lists every pattern that applies to its files.

```js
import { dirname } from "path";
import { fileURLToPath } from "url";
import { FlatCompat } from "@eslint/eslintrc";

const compat = new FlatCompat({ baseDirectory: dirname(fileURLToPath(import.meta.url)) });

const EVENTS_SCHEMA = {
  group: ["@/db/schema/events"],
  message: "Only modules/events touches the events table. Use logEvent() from @/modules/events.",
};
const DB_LAYER = {
  group: ["@/db", "@/db/*"],
  message: "app/ must go through a module in src/modules — never import the db layer directly.",
};
const CAMPAIGNS = {
  group: ["@/modules/campaigns", "@/modules/campaigns/*"],
  message: "Trust invariant (brief §0): trending/feed must not import campaign data.",
};

export default [
  ...compat.extends("next/core-web-vitals", "next/typescript"),
  {
    files: ["src/**/*.{ts,tsx}"],
    ignores: ["src/modules/events/**", "src/db/**"],
    rules: { "no-restricted-imports": ["error", { patterns: [EVENTS_SCHEMA] }] },
  },
  {
    files: ["src/app/**/*.{ts,tsx}"],
    rules: { "no-restricted-imports": ["error", { patterns: [EVENTS_SCHEMA, DB_LAYER] }] },
  },
  {
    files: ["src/modules/trending/**/*.{ts,tsx}", "src/modules/feed/**/*.{ts,tsx}"],
    rules: { "no-restricted-imports": ["error", { patterns: [EVENTS_SCHEMA, CAMPAIGNS] }] },
  },
  { ignores: [".next/", "node_modules/", "src/db/migrations/"] },
];
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm test tests/unit/eslint-boundaries.test.ts`
Expected: 6 passed.

- [ ] **Step 6: Run the full lint**

Run: `pnpm lint`
Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat: module skeleton with lint-enforced trust-invariant boundaries"
```

---

### Task 4: CI workflow

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Produces: CI running typecheck → lint → test (with Postgres 16 service) → build → bundle budget. Task 12 adds the budget script this references; until then the step is present but skipped via `if: hashFiles(...)`.

- [ ] **Step 1: Write `.github/workflows/ci.yml`**

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request:

jobs:
  ci:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: filmpadi
          POSTGRES_PASSWORD: filmpadi
          POSTGRES_DB: filmpadi_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U filmpadi"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 10 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
      - name: Bundle size budget
        if: ${{ hashFiles('scripts/check-bundle-size.mjs') != '' }}
        run: pnpm check:bundle
```

- [ ] **Step 2: Commit and push; verify the run**

```bash
git add -A && git commit -m "ci: typecheck, lint, test, build pipeline with Postgres service"
git push
```

Then check: `gh run watch --exit-status` (or the Actions tab). Expected: green.

---

### Task 5: Local Postgres 16 + databases + env files

**Files:**
- Create: `.env.local` (gitignored), `.env.test` (committed — local-only credentials, also matches the CI service)

**Interfaces:**
- Produces: `filmpadi_dev` and `filmpadi_test` databases reachable at `postgres://filmpadi:filmpadi@localhost:5432/…`; `DATABASE_URL` set for dev and tests.

- [ ] **Step 1: Install Postgres 16 (needs sudo — ask the human if running unattended)**

```bash
sudo apt update && sudo apt install -y postgresql postgresql-contrib
sudo service postgresql start
```

Verify: `psql --version` → `psql (PostgreSQL) 16.x`.
WSL note: `sudo service postgresql start` must be re-run after each Windows/WSL restart. Add this to the README in Step 5.

- [ ] **Step 2: Create role and databases**

```bash
sudo -u postgres psql -c "CREATE ROLE filmpadi LOGIN PASSWORD 'filmpadi' CREATEDB;"
sudo -u postgres createdb -O filmpadi filmpadi_dev
sudo -u postgres createdb -O filmpadi filmpadi_test
```

Verify: `PGPASSWORD=filmpadi psql -h localhost -U filmpadi -d filmpadi_dev -c "SELECT 1;"` → returns `1`.

- [ ] **Step 3: Write `.env.local`**

```
DATABASE_URL=postgres://filmpadi:filmpadi@localhost:5432/filmpadi_dev
SMS_PROVIDER=dev
STORAGE_PROVIDER=local
```

- [ ] **Step 4: Write `.env.test`**

```
DATABASE_URL=postgres://filmpadi:filmpadi@localhost:5432/filmpadi_test
SMS_PROVIDER=dev
STORAGE_PROVIDER=local
```

- [ ] **Step 5: Write `README.md`**

```markdown
# FilmPadi

Nollywood discovery platform — metadata only, never film files.

## Dev setup

1. Node 20 + pnpm 10, then `pnpm install`.
2. Postgres 16: `sudo service postgresql start` (required after every WSL restart).
   First time: create role `filmpadi` and dbs `filmpadi_dev` / `filmpadi_test`
   (see docs/superpowers/plans/2026-07-16-01-platform-substrate.md Task 5).
3. `cp .env.example .env.local` and set `DATABASE_URL` (default local values in the example work).
4. `pnpm db:migrate` then `pnpm dev`.

Docs: `docs/superpowers/specs/` (designs), `docs/superpowers/plans/` (implementation plans),
`docs/orban-forest-v1-technical-brief.md` (v1 brief; "Orban Forest" is the old codename).
```

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "chore: local Postgres env, test env, README dev setup"
```

---

### Task 6: Complete Drizzle schema + migrations

Entire brief §2 DDL plus spec additions (`sessions`, `otp_codes`, `rate_limits`, `youtube_channels`, FTS). Deviations from the brief's raw DDL, both noted in the spec or here: `follows` gets a surrogate `id` PK (its natural key contains NULLs) with partial unique indexes; the `search` tsvector lives in a hand-written second migration (generated columns + extensions are cleaner in SQL).

**Files:**
- Create: `drizzle.config.ts`, `src/db/client.ts`, `src/db/migrate.ts`, `src/db/schema/{index,identity,auth,catalog,producers,social,events,campaigns,moderation}.ts`, `tests/global-setup.ts`, `tests/helpers/db.ts`, `tests/integration/schema.test.ts`
- Modify: `vitest.config.ts` (add globalSetup), `src/db/.gitkeep` (delete)

**Interfaces:**
- Produces: `db` (drizzle instance) from `@/db/client`; all table objects from `@/db/schema` (index re-exports everything EXCEPT `events`); `events` table only from `@/db/schema/events`; `pnpm db:migrate` applies migrations to `$DATABASE_URL`; tests get a migrated `filmpadi_test` automatically via globalSetup.
- Consumes: databases from Task 5; boundary rules from Task 3.

- [ ] **Step 1: Write `drizzle.config.ts`**

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema/*.ts",
  out: "./src/db/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL ?? "" },
});
```

- [ ] **Step 2: Write `src/db/schema/identity.ts`**

```ts
import { pgTable, real, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  phone: text("phone").unique(), // E.164; null if OAuth-only
  email: text("email").unique(),
  role: text("role").notNull().default("viewer"), // viewer | producer | admin
  username: text("username").unique().notNull(),
  displayName: text("display_name"),
  avatarUrl: text("avatar_url"),
  bio: text("bio"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  phoneVerifiedAt: timestamp("phone_verified_at", { withTimezone: true }),
  trustWeight: real("trust_weight").notNull().default(0.2),
});
```

- [ ] **Step 3: Write `src/db/schema/auth.ts`**

```ts
import { bigint, index, integer, pgTable, primaryKey, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { users } from "./identity";

export const sessions = pgTable("sessions", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  tokenHash: text("token_hash").unique().notNull(), // sha256 of the cookie token
  expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const otpCodes = pgTable(
  "otp_codes",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    phone: text("phone").notNull(), // E.164
    codeHash: text("code_hash").notNull(),
    expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
    attempts: integer("attempts").notNull().default(0),
    consumedAt: timestamp("consumed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("otp_codes_phone_idx").on(t.phone)],
);

export const rateLimits = pgTable(
  "rate_limits",
  {
    scope: text("scope").notNull(), // e.g. 'otp-send-phone'
    identifier: text("identifier").notNull(), // phone, ip, user id…
    windowStart: bigint("window_start", { mode: "number" }).notNull(),
    count: integer("count").notNull().default(1),
  },
  (t) => [primaryKey({ columns: [t.scope, t.identifier, t.windowStart] })],
);
```

- [ ] **Step 4: Write `src/db/schema/catalog.ts`**

```ts
import { sql } from "drizzle-orm";
import { boolean, date, index, integer, pgTable, primaryKey, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const films = pgTable(
  "films",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    slug: text("slug").unique().notNull(),
    title: text("title").notNull(),
    releaseDate: date("release_date"), // drives the recency promise
    year: integer("year"),
    synopsis: text("synopsis"),
    posterUrl: text("poster_url"),
    runtimeMin: integer("runtime_min"),
    languages: text("languages").array().notNull().default(sql`'{}'::text[]`),
    genres: text("genres").array().notNull().default(sql`'{}'::text[]`),
    nfvcbRef: text("nfvcb_ref"),
    status: text("status").notNull().default("published"), // draft|published|flagged|removed
    source: text("source").notNull(), // nfvcb|youtube|producer|community|editorial
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("films_release_idx").on(t.releaseDate.desc()).where(sql`status = 'published'`)],
);

export const people = pgTable("people", {
  id: uuid("id").primaryKey().defaultRandom(),
  slug: text("slug").unique().notNull(),
  name: text("name").notNull(),
  photoUrl: text("photo_url"),
});

export const filmCredits = pgTable(
  "film_credits",
  {
    filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
    personId: uuid("person_id").notNull().references(() => people.id, { onDelete: "cascade" }),
    credit: text("credit").notNull(), // director|producer|cast|writer
    billing: integer("billing"),
  },
  (t) => [primaryKey({ columns: [t.filmId, t.personId, t.credit] })],
);

// "No link available" is itself demand signal: a published film with zero
// film_links rows is queryable on purpose (brief §2).
export const filmLinks = pgTable("film_links", {
  id: uuid("id").primaryKey().defaultRandom(),
  filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
  platform: text("platform").notNull(), // youtube|netflix|prime|cinema|other
  url: text("url").notNull(),
  label: text("label"),
  verified: boolean("verified").notNull().default(false),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const youtubeChannels = pgTable("youtube_channels", {
  id: uuid("id").primaryKey().defaultRandom(),
  channelId: text("channel_id").unique().notNull(),
  name: text("name").notNull(),
  addedAt: timestamp("added_at", { withTimezone: true }).notNull().defaultNow(),
});
```

- [ ] **Step 5: Write `src/db/schema/producers.ts`**

```ts
import { jsonb, pgTable, primaryKey, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { sql } from "drizzle-orm";
import { users } from "./identity";
import { films } from "./catalog";

export const producers = pgTable("producers", {
  id: uuid("id").primaryKey().defaultRandom(),
  slug: text("slug").unique().notNull(),
  name: text("name").notNull(),
  ownerUser: uuid("owner_user").references(() => users.id),
  verifiedAt: timestamp("verified_at", { withTimezone: true }),
  bio: text("bio"),
  logoUrl: text("logo_url"),
  socials: jsonb("socials").notNull().default(sql`'{}'::jsonb`),
});

export const producerFilms = pgTable(
  "producer_films",
  {
    producerId: uuid("producer_id").notNull().references(() => producers.id, { onDelete: "cascade" }),
    filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
    claimState: text("claim_state").notNull().default("pending"), // pending|approved|rejected
  },
  (t) => [primaryKey({ columns: [t.producerId, t.filmId] })],
);
```

- [ ] **Step 6: Write `src/db/schema/social.ts`**

```ts
import { sql } from "drizzle-orm";
import {
  boolean, check, integer, pgTable, primaryKey, real, text, timestamp, uniqueIndex, uuid,
} from "drizzle-orm/pg-core";
import { users } from "./identity";
import { films } from "./catalog";
import { producers } from "./producers";

export const ratings = pgTable(
  "ratings",
  {
    userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
    filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
    stars: real("stars").notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    primaryKey({ columns: [t.userId, t.filmId] }),
    check("ratings_stars_range", sql`stars BETWEEN 0.5 AND 5`),
  ],
);

export const reviews = pgTable("reviews", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
  body: text("body").notNull(), // tweet-length encouraged, 2000 char cap (enforced in app layer)
  spoiler: boolean("spoiler").notNull().default(false),
  status: text("status").notNull().default("live"), // live|flagged|removed
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const lists = pgTable(
  "lists",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
    slug: text("slug").notNull(),
    title: text("title").notNull(),
    description: text("description"),
    isRanked: boolean("is_ranked").notNull().default(false),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [uniqueIndex("lists_user_slug_uq").on(t.userId, t.slug)],
);

export const listItems = pgTable(
  "list_items",
  {
    listId: uuid("list_id").notNull().references(() => lists.id, { onDelete: "cascade" }),
    filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
    position: integer("position").notNull(),
    note: text("note"),
  },
  (t) => [primaryKey({ columns: [t.listId, t.filmId] })],
);

// Surrogate PK (deviation from brief DDL, which had no PK): the natural key
// contains NULLs. Partial unique indexes prevent duplicate follows.
export const follows = pgTable(
  "follows",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    follower: uuid("follower").notNull().references(() => users.id, { onDelete: "cascade" }),
    followedUser: uuid("followed_user").references(() => users.id, { onDelete: "cascade" }),
    followedProducer: uuid("followed_producer").references(() => producers.id, { onDelete: "cascade" }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    check("follows_exactly_one_target", sql`(followed_user IS NULL) <> (followed_producer IS NULL)`),
    uniqueIndex("follows_user_uq").on(t.follower, t.followedUser).where(sql`followed_user IS NOT NULL`),
    uniqueIndex("follows_producer_uq").on(t.follower, t.followedProducer).where(sql`followed_producer IS NOT NULL`),
  ],
);

export const watchIntent = pgTable(
  "watch_intent",
  {
    userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
    filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [primaryKey({ columns: [t.userId, t.filmId] })],
);

export const fourFavorites = pgTable(
  "four_favorites",
  {
    userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
    filmId: uuid("film_id").notNull().references(() => films.id, { onDelete: "cascade" }),
    position: integer("position").notNull(),
  },
  (t) => [
    primaryKey({ columns: [t.userId, t.position] }),
    check("four_favorites_position_range", sql`position BETWEEN 1 AND 4`),
  ],
);
```

- [ ] **Step 7: Write `src/db/schema/events.ts`** (imported ONLY by modules/events and db — lint-enforced)

```ts
import { bigserial, index, pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

// The taste graph's raw feed. campaign_id is ONLY set when surface='promoted';
// trending recompute reads WHERE campaign_id IS NULL (trust invariant, brief §7).
export const events = pgTable(
  "events",
  {
    id: bigserial("id", { mode: "number" }).primaryKey(),
    userId: uuid("user_id"), // null for anonymous
    anonId: text("anon_id"), // cookie id for anonymous
    type: text("type").notNull(), // film_view|outbound_click|watch_intent_add|rating|review|list_add|follow|share
    filmId: uuid("film_id"),
    producerId: uuid("producer_id"),
    surface: text("surface"), // home|new_releases|trending|feed|film_page|list|search|promoted
    campaignId: uuid("campaign_id"),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [index("events_film_time_idx").on(t.filmId, t.type, t.createdAt.desc())],
);
```

- [ ] **Step 8: Write `src/db/schema/campaigns.ts`**

```ts
import { sql } from "drizzle-orm";
import { bigint, check, jsonb, pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { producers } from "./producers";
import { films } from "./catalog";

export const wallets = pgTable(
  "wallets",
  {
    producerId: uuid("producer_id").primaryKey().references(() => producers.id),
    balanceKobo: bigint("balance_kobo", { mode: "bigint" }).notNull().default(sql`0`),
  },
  () => [check("wallets_balance_nonnegative", sql`balance_kobo >= 0`)],
);

export const walletTxns = pgTable("wallet_txns", {
  id: uuid("id").primaryKey().defaultRandom(),
  producerId: uuid("producer_id").notNull().references(() => producers.id),
  amountKobo: bigint("amount_kobo", { mode: "bigint" }).notNull(), // +topup, -spend
  kind: text("kind").notNull(), // topup|spend|refund
  paystackRef: text("paystack_ref"),
  campaignId: uuid("campaign_id"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const campaigns = pgTable("campaigns", {
  id: uuid("id").primaryKey().defaultRandom(),
  producerId: uuid("producer_id").notNull().references(() => producers.id),
  filmId: uuid("film_id").notNull().references(() => films.id),
  status: text("status").notNull().default("draft"), // draft|active|paused|exhausted|ended
  dailyBudgetKobo: bigint("daily_budget_kobo", { mode: "bigint" }).notNull(),
  totalBudgetKobo: bigint("total_budget_kobo", { mode: "bigint" }).notNull(),
  spentKobo: bigint("spent_kobo", { mode: "bigint" }).notNull().default(sql`0`),
  surfaces: text("surfaces").array().notNull().default(sql`'{home,genre_feed}'::text[]`), // NEVER 'trending'
  targeting: jsonb("targeting").notNull().default(sql`'{}'::jsonb`),
  startsAt: timestamp("starts_at", { withTimezone: true }),
  endsAt: timestamp("ends_at", { withTimezone: true }),
});
```

- [ ] **Step 9: Write `src/db/schema/moderation.ts`**

```ts
import { sql } from "drizzle-orm";
import { jsonb, pgTable, real, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { users } from "./identity";
import { films } from "./catalog";

export const ingestCandidates = pgTable("ingest_candidates", {
  id: uuid("id").primaryKey().defaultRandom(),
  source: text("source").notNull(), // nfvcb|youtube|producer|community
  raw: jsonb("raw").notNull(),
  matchedFilm: uuid("matched_film").references(() => films.id),
  state: text("state").notNull().default("new"), // new|merged|created|rejected
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const edits = pgTable("edits", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").references(() => users.id),
  filmId: uuid("film_id").references(() => films.id),
  patch: jsonb("patch").notNull(),
  state: text("state").notNull().default("pending"), // pending|approved|rejected|auto
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const flags = pgTable("flags", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id"),
  targetType: text("target_type").notNull(),
  targetId: uuid("target_id").notNull(),
  reason: text("reason"),
  state: text("state").notNull().default("open"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const trendingScores = pgTable("trending_scores", {
  filmId: uuid("film_id").primaryKey().references(() => films.id),
  score: real("score").notNull(),
  windowSignals: jsonb("window_signals").notNull(), // per-signal velocities, for forensics
  editorialState: text("editorial_state").notNull().default("auto"), // auto|approved|held
  computedAt: timestamp("computed_at", { withTimezone: true }).notNull(),
});
```

- [ ] **Step 10: Write `src/db/schema/index.ts`** (re-exports everything EXCEPT events — boundary by omission)

```ts
export * from "./identity";
export * from "./auth";
export * from "./catalog";
export * from "./producers";
export * from "./social";
export * from "./campaigns";
export * from "./moderation";
// NOTE: ./events is deliberately NOT re-exported. Import it only from
// src/modules/events (lint-enforced).
```

- [ ] **Step 11: Write `src/db/client.ts` and `src/db/migrate.ts`**

```ts
// src/db/client.ts
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "./schema";

const url = process.env.DATABASE_URL;
if (!url) throw new Error("DATABASE_URL is not set");

const globalForDb = globalThis as unknown as { pgClient?: ReturnType<typeof postgres> };
export const sql = globalForDb.pgClient ?? postgres(url);
if (process.env.NODE_ENV !== "production") globalForDb.pgClient = sql;

export const db = drizzle(sql, { schema });
```

```ts
// src/db/migrate.ts
import { drizzle } from "drizzle-orm/postgres-js";
import { migrate } from "drizzle-orm/postgres-js/migrator";
import postgres from "postgres";

async function main() {
  const url = process.env.DATABASE_URL;
  if (!url) throw new Error("DATABASE_URL is not set");
  const client = postgres(url, { max: 1 });
  await migrate(drizzle(client), { migrationsFolder: "src/db/migrations" });
  await client.end();
  console.log("migrations applied");
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

- [ ] **Step 12: Generate migration 0001 and hand-write the FTS migration**

```bash
rm src/db/.gitkeep
DATABASE_URL=postgres://filmpadi:filmpadi@localhost:5432/filmpadi_dev pnpm db:generate
pnpm drizzle-kit generate --custom --name=films-fts
```

Then fill the generated empty custom migration (`src/db/migrations/0001_films-fts.sql` or similar) with:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

ALTER TABLE films ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('simple', coalesce(synopsis, '')), 'B')
  ) STORED;

CREATE INDEX films_search_idx ON films USING gin (search);
CREATE INDEX films_title_trgm_idx ON films USING gin (title gin_trgm_ops);
```

- [ ] **Step 13: Write the failing schema test**

`tests/helpers/db.ts`:

```ts
import postgres from "postgres";

export function testSql() {
  const url = process.env.DATABASE_URL;
  if (!url || !url.includes("filmpadi_test")) {
    throw new Error(`Tests must run against filmpadi_test, got: ${url}`);
  }
  return postgres(url, { max: 2 });
}
```

`tests/global-setup.ts` (migrates the test db once per run; drizzle's journal makes it idempotent):

```ts
import { config } from "dotenv";
import { drizzle } from "drizzle-orm/postgres-js";
import { migrate } from "drizzle-orm/postgres-js/migrator";
import postgres from "postgres";

export default async function setup() {
  config({ path: ".env.test" });
  const url = process.env.DATABASE_URL;
  if (!url || !url.includes("filmpadi_test")) {
    throw new Error(`global-setup: expected filmpadi_test DATABASE_URL, got: ${url}`);
  }
  const client = postgres(url, { max: 1 });
  await migrate(drizzle(client), { migrationsFolder: "src/db/migrations" });
  await client.end();
}
```

Add to `vitest.config.ts` inside `test: {}`: `globalSetup: ["tests/global-setup.ts"],`

`tests/integration/schema.test.ts`:

```ts
import { afterAll, describe, expect, it } from "vitest";
import { testSql } from "../helpers/db";

const sql = testSql();
afterAll(() => sql.end());

const EXPECTED_TABLES = [
  "users", "sessions", "otp_codes", "rate_limits",
  "films", "people", "film_credits", "film_links", "youtube_channels",
  "producers", "producer_films",
  "ratings", "reviews", "lists", "list_items", "follows", "watch_intent", "four_favorites",
  "events", "wallets", "wallet_txns", "campaigns",
  "ingest_candidates", "edits", "flags", "trending_scores",
];

describe("schema migration", () => {
  it("creates every table from the brief + spec", async () => {
    const rows = await sql`SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'`;
    const names = rows.map((r) => r.table_name as string);
    for (const t of EXPECTED_TABLES) expect(names, `missing table ${t}`).toContain(t);
  });

  it("adds the FTS generated column and trigram extension", async () => {
    const col = await sql`SELECT 1 FROM information_schema.columns WHERE table_name = 'films' AND column_name = 'search'`;
    expect(col.length).toBe(1);
    const ext = await sql`SELECT 1 FROM pg_extension WHERE extname = 'pg_trgm'`;
    expect(ext.length).toBe(1);
  });

  it("rejects ratings outside 0.5–5", async () => {
    await sql`INSERT INTO users (username) VALUES ('schema_test_user') ON CONFLICT DO NOTHING`;
    await sql`INSERT INTO films (slug, title, source) VALUES ('schema-test-film', 'Schema Test', 'editorial') ON CONFLICT DO NOTHING`;
    await expect(
      sql`INSERT INTO ratings (user_id, film_id, stars)
          SELECT u.id, f.id, 6 FROM users u, films f
          WHERE u.username = 'schema_test_user' AND f.slug = 'schema-test-film'`,
    ).rejects.toThrow(/ratings_stars_range/);
  });

  it("rejects follows with both or neither target (XOR check)", async () => {
    await expect(
      sql`INSERT INTO follows (follower)
          SELECT id FROM users WHERE username = 'schema_test_user'`,
    ).rejects.toThrow(/follows_exactly_one_target/);
  });
});
```

- [ ] **Step 14: Run the schema tests**

Run: `pnpm test tests/integration/schema.test.ts`
Expected: PASS — globalSetup migrates `filmpadi_test` (including the FTS migration), then all four assertions hold. (The fail-first checkpoint for this task is Step 13 run before Step 12's migrations exist, if you executed out of order; with migrations in place these must pass.)

- [ ] **Step 15: Make `pnpm db:migrate` load `.env.local`, then migrate the dev db**

`tsx` does not auto-load `.env.local`, so add `dotenv-cli` and update the script:

```bash
pnpm add -D dotenv-cli
```

In `package.json`, change the script to:

```json
"db:migrate": "dotenv -e .env.local -- tsx src/db/migrate.ts"
```

(CI and tests are unaffected — they set `DATABASE_URL` explicitly / via globalSetup.)

Run: `pnpm db:migrate`
Expected: `migrations applied`.

Run: `pnpm test`
Expected: all pass (boundary tests, smoke, schema).

- [ ] **Step 16: Typecheck, lint, commit**

```bash
pnpm typecheck && pnpm lint
git add -A && git commit -m "feat: complete Drizzle schema (brief §2 + auth/rate-limit/FTS) with migrations and tests"
```

---

### Task 7: `lib/errors` + `lib/config`

**Files:**
- Create: `src/lib/errors.ts`, `src/lib/config.ts`, `tests/unit/errors.test.ts`, `tests/unit/config.test.ts`

**Interfaces:**
- Produces: `AppError`, `NotFoundError`, `ValidationError`, `RateLimitError`, `AuthError` (all with `.code: string`, `.status: number`) from `@/lib/errors`; `getConfig(): Config` from `@/lib/config` where `Config = { smsProvider: "dev" | "termii"; storageProvider: "local" | "r2"; termiiApiKey?: string; r2: {...} | undefined }`. `resetConfigForTests()` clears the memo.

- [ ] **Step 1: Write failing tests — `tests/unit/errors.test.ts`**

```ts
import { describe, expect, it } from "vitest";
import { AppError, AuthError, NotFoundError, RateLimitError, ValidationError } from "@/lib/errors";

describe("error hierarchy", () => {
  it("carries code and http status", () => {
    expect(new NotFoundError("film").status).toBe(404);
    expect(new ValidationError("bad phone").status).toBe(400);
    expect(new RateLimitError().status).toBe(429);
    expect(new AuthError().status).toBe(401);
  });

  it("all extend AppError with stable codes", () => {
    const e = new NotFoundError("film");
    expect(e).toBeInstanceOf(AppError);
    expect(e.code).toBe("not_found");
    expect(e.message).toContain("film");
  });
});
```

- [ ] **Step 2: Write failing tests — `tests/unit/config.test.ts`**

```ts
import { afterEach, describe, expect, it } from "vitest";
import { getConfig, resetConfigForTests } from "@/lib/config";

afterEach(() => {
  resetConfigForTests();
  delete process.env.SMS_PROVIDER;
  delete process.env.TERMII_API_KEY;
});

describe("config", () => {
  it("defaults to dev sms and local storage", () => {
    expect(getConfig().smsProvider).toBe("dev");
    expect(getConfig().storageProvider).toBe("local");
  });

  it("requires TERMII_API_KEY when SMS_PROVIDER=termii", () => {
    process.env.SMS_PROVIDER = "termii";
    expect(() => getConfig()).toThrow(/TERMII_API_KEY/);
  });

  it("accepts termii when the key is present", () => {
    process.env.SMS_PROVIDER = "termii";
    process.env.TERMII_API_KEY = "tk_test";
    expect(getConfig().smsProvider).toBe("termii");
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `pnpm test tests/unit/errors.test.ts tests/unit/config.test.ts`
Expected: FAIL — modules do not exist.

- [ ] **Step 4: Write `src/lib/errors.ts`**

```ts
export class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly status: number,
  ) {
    super(message);
    this.name = new.target.name;
  }
}

export class NotFoundError extends AppError {
  constructor(what = "resource") {
    super(`${what} not found`, "not_found", 404);
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, "validation_error", 400);
  }
}

export class RateLimitError extends AppError {
  constructor(message = "too many requests, slow down") {
    super(message, "rate_limited", 429);
  }
}

export class AuthError extends AppError {
  constructor(message = "not signed in") {
    super(message, "unauthorized", 401);
  }
}
```

- [ ] **Step 5: Write `src/lib/config.ts`**

```ts
import { z } from "zod";

const envSchema = z
  .object({
    SMS_PROVIDER: z.enum(["dev", "termii"]).default("dev"),
    STORAGE_PROVIDER: z.enum(["local", "r2"]).default("local"),
    TERMII_API_KEY: z.string().optional(),
    R2_ACCOUNT_ID: z.string().optional(),
    R2_ACCESS_KEY: z.string().optional(),
    R2_SECRET: z.string().optional(),
    R2_BUCKET: z.string().optional(),
    R2_PUBLIC_BASE_URL: z.string().optional(),
  })
  .superRefine((env, ctx) => {
    if (env.SMS_PROVIDER === "termii" && !env.TERMII_API_KEY) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, message: "TERMII_API_KEY is required when SMS_PROVIDER=termii" });
    }
    if (env.STORAGE_PROVIDER === "r2") {
      for (const k of ["R2_ACCOUNT_ID", "R2_ACCESS_KEY", "R2_SECRET", "R2_BUCKET", "R2_PUBLIC_BASE_URL"] as const) {
        if (!env[k]) ctx.addIssue({ code: z.ZodIssueCode.custom, message: `${k} is required when STORAGE_PROVIDER=r2` });
      }
    }
  });

export type Config = {
  smsProvider: "dev" | "termii";
  storageProvider: "local" | "r2";
  termiiApiKey?: string;
  r2?: { accountId: string; accessKey: string; secret: string; bucket: string; publicBaseUrl: string };
};

let memo: Config | undefined;

export function getConfig(): Config {
  if (memo) return memo;
  const env = envSchema.parse(process.env);
  memo = {
    smsProvider: env.SMS_PROVIDER,
    storageProvider: env.STORAGE_PROVIDER,
    termiiApiKey: env.TERMII_API_KEY,
    r2:
      env.STORAGE_PROVIDER === "r2"
        ? {
            accountId: env.R2_ACCOUNT_ID!,
            accessKey: env.R2_ACCESS_KEY!,
            secret: env.R2_SECRET!,
            bucket: env.R2_BUCKET!,
            publicBaseUrl: env.R2_PUBLIC_BASE_URL!,
          }
        : undefined,
  };
  return memo;
}

export function resetConfigForTests() {
  memo = undefined;
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `pnpm test tests/unit/errors.test.ts tests/unit/config.test.ts`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat: typed error hierarchy and validated env config"
```

---

### Task 8: Postgres-backed rate limiter

Fixed-window counter in the `rate_limits` table (no Redis in the $50 envelope; a table is fine at this scale). Used by auth (Plan 2) and all write endpoints later.

**Files:**
- Create: `src/lib/rate-limit.ts`, `tests/integration/rate-limit.test.ts`

**Interfaces:**
- Consumes: `rateLimits` table from `@/db/schema` (Task 6), `RateLimitError` from `@/lib/errors` (Task 7).
- Produces: `consumeRateLimit(scope: string, identifier: string, opts: { limit: number; windowSeconds: number }): Promise<void>` from `@/lib/rate-limit` — resolves if within limit, throws `RateLimitError` if over.

- [ ] **Step 1: Write the failing test — `tests/integration/rate-limit.test.ts`**

```ts
import { describe, expect, it } from "vitest";
import { consumeRateLimit } from "@/lib/rate-limit";
import { RateLimitError } from "@/lib/errors";

describe("consumeRateLimit", () => {
  it("allows up to the limit then throws RateLimitError", async () => {
    const id = `test-${Math.random().toString(36).slice(2)}`;
    await consumeRateLimit("test-scope", id, { limit: 3, windowSeconds: 3600 });
    await consumeRateLimit("test-scope", id, { limit: 3, windowSeconds: 3600 });
    await consumeRateLimit("test-scope", id, { limit: 3, windowSeconds: 3600 });
    await expect(consumeRateLimit("test-scope", id, { limit: 3, windowSeconds: 3600 })).rejects.toBeInstanceOf(
      RateLimitError,
    );
  });

  it("tracks identifiers independently", async () => {
    const a = `test-${Math.random().toString(36).slice(2)}`;
    const b = `test-${Math.random().toString(36).slice(2)}`;
    await consumeRateLimit("test-scope", a, { limit: 1, windowSeconds: 3600 });
    await expect(consumeRateLimit("test-scope", b, { limit: 1, windowSeconds: 3600 })).resolves.toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test tests/integration/rate-limit.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Write `src/lib/rate-limit.ts`**

```ts
import { sql as dsql } from "drizzle-orm";
import { db } from "@/db/client";
import { rateLimits } from "@/db/schema";
import { RateLimitError } from "@/lib/errors";

export async function consumeRateLimit(
  scope: string,
  identifier: string,
  opts: { limit: number; windowSeconds: number },
): Promise<void> {
  const windowStart = Math.floor(Date.now() / 1000 / opts.windowSeconds) * opts.windowSeconds;

  const rows = await db
    .insert(rateLimits)
    .values({ scope, identifier, windowStart, count: 1 })
    .onConflictDoUpdate({
      target: [rateLimits.scope, rateLimits.identifier, rateLimits.windowStart],
      set: { count: dsql`${rateLimits.count} + 1` },
    })
    .returning({ count: rateLimits.count });

  const count = rows[0]?.count ?? Number.MAX_SAFE_INTEGER;
  if (count > opts.limit) throw new RateLimitError();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm test tests/integration/rate-limit.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: Postgres-backed fixed-window rate limiter"
```

---

### Task 9: SMS adapter (Dev + Termii)

**Files:**
- Create: `src/lib/adapters/sms/types.ts`, `src/lib/adapters/sms/dev.ts`, `src/lib/adapters/sms/termii.ts`, `src/lib/adapters/sms/index.ts`, `tests/unit/sms.test.ts`

**Interfaces:**
- Consumes: `getConfig()` from Task 7.
- Produces: `SmsProvider` interface `{ sendOtp(phoneE164: string, code: string): Promise<void> }`; `getSmsProvider(): SmsProvider` from `@/lib/adapters/sms` (selects by config); `DevSms`, `TermiiSms` classes. Plan 2's OTP service consumes `getSmsProvider()`.

- [ ] **Step 1: Write the failing test — `tests/unit/sms.test.ts`**

```ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { DevSms } from "@/lib/adapters/sms/dev";
import { TermiiSms } from "@/lib/adapters/sms/termii";
import { getSmsProvider } from "@/lib/adapters/sms";
import { resetConfigForTests } from "@/lib/config";

afterEach(() => {
  resetConfigForTests();
  delete process.env.SMS_PROVIDER;
  delete process.env.TERMII_API_KEY;
  vi.restoreAllMocks();
});

describe("DevSms", () => {
  it("logs the code instead of sending", async () => {
    const info = vi.spyOn(console, "info").mockImplementation(() => {});
    await new DevSms().sendOtp("+2348012345678", "123456");
    expect(info).toHaveBeenCalledWith(expect.stringContaining("123456"));
  });
});

describe("TermiiSms", () => {
  it("POSTs to the Termii sms endpoint with the api key and code", async () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response(JSON.stringify({ code: "ok" }), { status: 200 }));
    await new TermiiSms("tk_test", "FilmPadi", fetchFn as unknown as typeof fetch).sendOtp("+2348012345678", "123456");
    const [url, init] = fetchFn.mock.calls[0]!;
    expect(url).toContain("termii.com");
    const body = JSON.parse((init as RequestInit).body as string);
    expect(body.api_key).toBe("tk_test");
    expect(body.to).toBe("+2348012345678");
    expect(body.sms).toContain("123456");
  });

  it("throws on non-2xx responses", async () => {
    const fetchFn = vi.fn().mockResolvedValue(new Response("nope", { status: 401 }));
    await expect(
      new TermiiSms("tk_bad", "FilmPadi", fetchFn as unknown as typeof fetch).sendOtp("+2348012345678", "123456"),
    ).rejects.toThrow(/termii/i);
  });
});

describe("getSmsProvider", () => {
  it("returns DevSms by default", () => {
    expect(getSmsProvider()).toBeInstanceOf(DevSms);
  });

  it("returns TermiiSms when configured", () => {
    process.env.SMS_PROVIDER = "termii";
    process.env.TERMII_API_KEY = "tk_test";
    expect(getSmsProvider()).toBeInstanceOf(TermiiSms);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test tests/unit/sms.test.ts`
Expected: FAIL — modules do not exist.

- [ ] **Step 3: Implement the adapter**

`src/lib/adapters/sms/types.ts`:

```ts
export interface SmsProvider {
  sendOtp(phoneE164: string, code: string): Promise<void>;
}
```

`src/lib/adapters/sms/dev.ts`:

```ts
import type { SmsProvider } from "./types";

export class DevSms implements SmsProvider {
  async sendOtp(phoneE164: string, code: string): Promise<void> {
    console.info(`[dev-sms] OTP for ${phoneE164}: ${code}`);
  }
}
```

`src/lib/adapters/sms/termii.ts`:

```ts
import type { SmsProvider } from "./types";

const TERMII_SMS_URL = "https://api.ng.termii.com/api/sms/send";

export class TermiiSms implements SmsProvider {
  constructor(
    private readonly apiKey: string,
    private readonly senderId = "FilmPadi",
    private readonly fetchFn: typeof fetch = fetch,
  ) {}

  async sendOtp(phoneE164: string, code: string): Promise<void> {
    const res = await this.fetchFn(TERMII_SMS_URL, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({
        api_key: this.apiKey,
        to: phoneE164,
        from: this.senderId,
        sms: `Your FilmPadi code is ${code}. It expires in 10 minutes.`,
        type: "plain",
        channel: "dnd",
      }),
    });
    if (!res.ok) {
      throw new Error(`termii send failed: ${res.status} ${await res.text()}`);
    }
  }
}
```

`src/lib/adapters/sms/index.ts`:

```ts
import { getConfig } from "@/lib/config";
import { DevSms } from "./dev";
import { TermiiSms } from "./termii";
import type { SmsProvider } from "./types";

export type { SmsProvider };
export { DevSms, TermiiSms };

export function getSmsProvider(): SmsProvider {
  const config = getConfig();
  if (config.smsProvider === "termii") return new TermiiSms(config.termiiApiKey!);
  return new DevSms();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm test tests/unit/sms.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: SMS adapter with dev and Termii providers"
```

---

### Task 10: Image storage adapter (Local + R2)

**Files:**
- Create: `src/lib/adapters/storage/types.ts`, `src/lib/adapters/storage/local.ts`, `src/lib/adapters/storage/r2.ts`, `src/lib/adapters/storage/index.ts`, `tests/unit/storage.test.ts`
- Modify: `package.json` (add `@aws-sdk/client-s3` — R2 is S3-compatible)

**Interfaces:**
- Consumes: `getConfig()` from Task 7.
- Produces: `ImageStorage` interface `{ put(key: string, data: Uint8Array, contentType: string): Promise<{ url: string }> }`; `getImageStorage(): ImageStorage` from `@/lib/adapters/storage`. Poster/avatar upload code in later plans consumes this.

- [ ] **Step 1: Add the S3 client**

Run: `pnpm add @aws-sdk/client-s3`

- [ ] **Step 2: Write the failing test — `tests/unit/storage.test.ts`**

```ts
import { mkdtempSync, readFileSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { afterAll, describe, expect, it, vi } from "vitest";
import { LocalImageStorage } from "@/lib/adapters/storage/local";
import { R2ImageStorage } from "@/lib/adapters/storage/r2";

const tmp = mkdtempSync(join(tmpdir(), "filmpadi-storage-"));
afterAll(() => rmSync(tmp, { recursive: true, force: true }));

describe("LocalImageStorage", () => {
  it("writes the file and returns a /uploads url", async () => {
    const storage = new LocalImageStorage(tmp, "/uploads");
    const { url } = await storage.put("posters/test.png", new Uint8Array([1, 2, 3]), "image/png");
    expect(url).toBe("/uploads/posters/test.png");
    expect([...readFileSync(join(tmp, "posters/test.png"))]).toEqual([1, 2, 3]);
  });

  it("rejects path-traversal keys", async () => {
    const storage = new LocalImageStorage(tmp, "/uploads");
    await expect(storage.put("../evil.png", new Uint8Array([1]), "image/png")).rejects.toThrow(/key/i);
  });
});

describe("R2ImageStorage", () => {
  it("sends a PutObject and returns the public url", async () => {
    const send = vi.fn().mockResolvedValue({});
    const storage = new R2ImageStorage(
      { send } as never,
      "filmpadi-images",
      "https://images.filmpadi.example",
    );
    const { url } = await storage.put("posters/test.png", new Uint8Array([1]), "image/png");
    expect(url).toBe("https://images.filmpadi.example/posters/test.png");
    expect(send).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm test tests/unit/storage.test.ts`
Expected: FAIL — modules do not exist.

- [ ] **Step 4: Implement the adapter**

`src/lib/adapters/storage/types.ts`:

```ts
export interface ImageStorage {
  put(key: string, data: Uint8Array, contentType: string): Promise<{ url: string }>;
}
```

`src/lib/adapters/storage/local.ts`:

```ts
import { mkdir, writeFile } from "node:fs/promises";
import { dirname, join, normalize } from "node:path";
import type { ImageStorage } from "./types";

export class LocalImageStorage implements ImageStorage {
  constructor(
    private readonly baseDir = "public/uploads",
    private readonly publicPrefix = "/uploads",
  ) {}

  async put(key: string, data: Uint8Array, _contentType: string): Promise<{ url: string }> {
    const clean = normalize(key);
    if (clean.startsWith("..") || clean.startsWith("/")) throw new Error(`invalid storage key: ${key}`);
    const path = join(this.baseDir, clean);
    await mkdir(dirname(path), { recursive: true });
    await writeFile(path, data);
    return { url: `${this.publicPrefix}/${clean}` };
  }
}
```

`src/lib/adapters/storage/r2.ts`:

```ts
import { PutObjectCommand, S3Client } from "@aws-sdk/client-s3";
import type { ImageStorage } from "./types";

export class R2ImageStorage implements ImageStorage {
  constructor(
    private readonly client: S3Client,
    private readonly bucket: string,
    private readonly publicBaseUrl: string,
  ) {}

  async put(key: string, data: Uint8Array, contentType: string): Promise<{ url: string }> {
    await this.client.send(
      new PutObjectCommand({ Bucket: this.bucket, Key: key, Body: data, ContentType: contentType }),
    );
    return { url: `${this.publicBaseUrl}/${key}` };
  }
}

export function r2Client(accountId: string, accessKey: string, secret: string): S3Client {
  return new S3Client({
    region: "auto",
    endpoint: `https://${accountId}.r2.cloudflarestorage.com`,
    credentials: { accessKeyId: accessKey, secretAccessKey: secret },
  });
}
```

`src/lib/adapters/storage/index.ts`:

```ts
import { getConfig } from "@/lib/config";
import { LocalImageStorage } from "./local";
import { R2ImageStorage, r2Client } from "./r2";
import type { ImageStorage } from "./types";

export type { ImageStorage };
export { LocalImageStorage, R2ImageStorage };

export function getImageStorage(): ImageStorage {
  const config = getConfig();
  if (config.storageProvider === "r2" && config.r2) {
    const { accountId, accessKey, secret, bucket, publicBaseUrl } = config.r2;
    return new R2ImageStorage(r2Client(accountId, accessKey, secret), bucket, publicBaseUrl);
  }
  return new LocalImageStorage();
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm test tests/unit/storage.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: image storage adapter with local and R2 providers"
```

---

### Task 11: Design tokens + base layout

The FilmPadi shell every page renders inside: dark cinematic ground, green accent, single variable font, designed at 360px first.

**Files:**
- Create: `src/components/site-header.tsx`, `src/components/site-footer.tsx`, `src/app/not-found.tsx`, `src/app/error.tsx`, `tests/unit/site-header.test.tsx`
- Modify: `src/app/globals.css`, `src/app/layout.tsx`, `src/app/page.tsx`

**Interfaces:**
- Produces: design tokens as Tailwind theme variables (`bg`, `surface`, `surface-raised`, `ink`, `ink-muted`, `accent`, `accent-strong`); `<SiteHeader />`, `<SiteFooter />` components. Plans 2–3 build pages inside this shell using these tokens only — no ad-hoc colors.

- [ ] **Step 1: Write the failing test — `tests/unit/site-header.test.tsx`**

```tsx
// @vitest-environment jsdom
import { render, screen } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import { SiteHeader } from "@/components/site-header";

describe("SiteHeader", () => {
  it("renders the FilmPadi wordmark linking home", () => {
    render(<SiteHeader />);
    const link = screen.getByRole("link", { name: /filmpadi/i });
    expect(link.getAttribute("href")).toBe("/");
  });

  it("renders the search link", () => {
    render(<SiteHeader />);
    expect(screen.getByRole("link", { name: /search/i }).getAttribute("href")).toBe("/search");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test tests/unit/site-header.test.tsx`
Expected: FAIL — component does not exist.

- [ ] **Step 3: Write `src/app/globals.css`** (replace entirely)

```css
@import "tailwindcss";

@theme {
  /* FilmPadi tokens — dark cinematic, poster-forward, green accent */
  --color-bg: #14120f; /* warm near-black ground */
  --color-surface: #1e1b17; /* cards, rails */
  --color-surface-raised: #2a2620; /* hover, modals */
  --color-ink: #f2efe9; /* primary text */
  --color-ink-muted: #9a938a; /* secondary text */
  --color-accent: #31c46c; /* interactive green (Nigeria's color) */
  --color-accent-strong: #1fa457; /* pressed/active */
  --color-danger: #e5484d;

  --font-sans: var(--font-space-grotesk), system-ui, sans-serif;
}

body {
  background-color: var(--color-bg);
  color: var(--color-ink);
}
```

- [ ] **Step 4: Write `src/components/site-header.tsx`**

```tsx
import Link from "next/link";

export function SiteHeader() {
  return (
    <header className="sticky top-0 z-40 border-b border-surface-raised bg-bg/95 backdrop-blur">
      <div className="mx-auto flex h-14 max-w-5xl items-center justify-between px-4">
        <Link href="/" className="text-lg font-bold tracking-tight text-ink">
          Film<span className="text-accent">Padi</span>
        </Link>
        <nav className="flex items-center gap-4 text-sm text-ink-muted">
          <Link href="/search" className="hover:text-ink">
            Search
          </Link>
        </nav>
      </div>
    </header>
  );
}
```

- [ ] **Step 5: Write `src/components/site-footer.tsx`**

```tsx
export function SiteFooter() {
  return (
    <footer className="mt-16 border-t border-surface-raised py-8 text-center text-xs text-ink-muted">
      <p>FilmPadi — the complete, current picture of Nollywood.</p>
    </footer>
  );
}
```

- [ ] **Step 6: Update `src/app/layout.tsx`** (font + shell; `next/font` self-hosts at build — no runtime third-party request)

```tsx
import type { Metadata } from "next";
import { Space_Grotesk } from "next/font/google";
import { SiteHeader } from "@/components/site-header";
import { SiteFooter } from "@/components/site-footer";
import "./globals.css";

const spaceGrotesk = Space_Grotesk({
  subsets: ["latin"],
  variable: "--font-space-grotesk",
});

export const metadata: Metadata = {
  title: { default: "FilmPadi", template: "%s — FilmPadi" },
  description: "The complete, current picture of Nollywood — new releases, where to watch, reviews.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={spaceGrotesk.variable}>
      <body className="font-sans antialiased">
        <SiteHeader />
        <div className="mx-auto min-h-[70vh] max-w-5xl px-4">{children}</div>
        <SiteFooter />
      </body>
    </html>
  );
}
```

- [ ] **Step 7: Update `src/app/page.tsx`** (placeholder home; the real recent-releases grid arrives in Plan 3)

```tsx
export default function HomePage() {
  return (
    <main className="py-10">
      <h1 className="text-2xl font-bold">New in Nollywood</h1>
      <p className="mt-2 text-ink-muted">The catalog is on its way. Recent releases land here.</p>
    </main>
  );
}
```

- [ ] **Step 8: Write `src/app/not-found.tsx` and `src/app/error.tsx`**

```tsx
// src/app/not-found.tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <main className="flex min-h-[50vh] flex-col items-center justify-center gap-3 text-center">
      <h1 className="text-3xl font-bold">Not found</h1>
      <p className="text-ink-muted">That page doesn&apos;t exist — maybe the film was removed.</p>
      <Link href="/" className="text-accent hover:text-accent-strong">
        Back to FilmPadi
      </Link>
    </main>
  );
}
```

```tsx
// src/app/error.tsx
"use client";

export default function ErrorPage({ reset }: { error: Error; reset: () => void }) {
  return (
    <main className="flex min-h-[50vh] flex-col items-center justify-center gap-3 text-center">
      <h1 className="text-3xl font-bold">Something broke</h1>
      <p className="text-ink-muted">Not your fault. Try again.</p>
      <button onClick={reset} className="rounded bg-accent px-4 py-2 font-medium text-bg hover:bg-accent-strong">
        Retry
      </button>
    </main>
  );
}
```

- [ ] **Step 9: Run tests and visual check**

Run: `pnpm test tests/unit/site-header.test.tsx` → PASS.
Run: `pnpm dev` and view http://localhost:3000 at 360px width in devtools — dark shell, wordmark, placeholder copy. Kill the server after.

- [ ] **Step 10: Commit**

```bash
git add -A && git commit -m "feat: FilmPadi design tokens and base layout shell"
```

---

### Task 12: Bundle-size budget check

**Files:**
- Create: `scripts/check-bundle-size.mjs`

**Interfaces:**
- Consumes: `.next/app-build-manifest.json` from `pnpm build`; the CI step from Task 4 (already wired, currently skipped) activates when this file lands.
- Produces: `pnpm check:bundle` — exits 1 if any viewer route exceeds 150KB gzipped first-load JS.

- [ ] **Step 1: Write `scripts/check-bundle-size.mjs`**

```js
// Enforces brief §3a: first-load JS < 150KB gzipped on viewer routes.
// Run after `pnpm build`.
import { readFileSync } from "node:fs";
import { gzipSync } from "node:zlib";

const LIMIT_BYTES = 150 * 1024;
// Non-viewer surfaces are exempt (producer/admin tooling can be heavier).
const EXEMPT = [/^\/studio/, /^\/admin/, /^\/api\//];

const manifest = JSON.parse(readFileSync(".next/app-build-manifest.json", "utf8"));

let failed = false;
for (const [route, files] of Object.entries(manifest.pages)) {
  const publicRoute = route.replace(/\/page$/, "") || "/";
  if (EXEMPT.some((re) => re.test(publicRoute))) continue;
  const gzipped = files
    .filter((f) => f.endsWith(".js"))
    .reduce((sum, f) => sum + gzipSync(readFileSync(`.next/${f}`)).length, 0);
  const kb = (gzipped / 1024).toFixed(1);
  const ok = gzipped <= LIMIT_BYTES;
  console.log(`${ok ? "  OK" : "FAIL"}  ${publicRoute.padEnd(32)} ${kb} KB gzipped`);
  if (!ok) failed = true;
}

if (failed) {
  console.error(`\nBudget exceeded: viewer routes must stay under ${LIMIT_BYTES / 1024} KB gzipped first-load JS.`);
  process.exit(1);
}
console.log("\nBundle budget OK.");
```

- [ ] **Step 2: Run it against a fresh build**

Run: `pnpm build && pnpm check:bundle`
Expected: all routes `OK`, exit 0. (The placeholder home should be far under budget; if not, something is badly wrong with the scaffold.)

- [ ] **Step 3: Commit and push; verify CI runs the budget step**

```bash
git add -A && git commit -m "feat: CI-enforced 150KB first-load JS budget for viewer routes"
git push
```

Check `gh run watch --exit-status` — the "Bundle size budget" step must now execute (no longer skipped) and pass.

---

## Done criteria for Plan 1

- `pnpm typecheck && pnpm lint && pnpm test && pnpm build && pnpm check:bundle` all green locally and in CI.
- `filmpadi_dev` and `filmpadi_test` are migrated with all 26 tables + FTS.
- Boundary lint rules are live and unit-tested.
- http://localhost:3000 shows the FilmPadi shell.

**Next plans (written after this one executes):**
- **Plan 2 — Auth:** OTP send/verify service + sessions + guards + username onboarding + Google OAuth (dormant), consuming `getSmsProvider()`, `consumeRateLimit()`, and the `sessions`/`otp_codes` tables from this plan.
- **Plan 3 — Catalog, SEO, events, search:** seed data, film/person/producer pages, `/out` tracking, JSON-LD, sitemap, OG cards, search, real home page.



