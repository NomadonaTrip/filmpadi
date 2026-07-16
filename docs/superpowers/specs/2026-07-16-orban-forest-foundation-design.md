# Orban Forest — Foundation Phase Design (Weeks 1–2)

**Date:** 2026-07-16
**Source:** [v1 Technical Brief](../../orban-forest-v1-technical-brief.md) (Weeks 1–2 of the build sequence)
**Status:** Approved design, pending implementation plan

## Context and scope

Orban Forest v1 is a web-based Nollywood discovery platform: metadata only, never film files. The full v1 is an 8-week build; this spec covers the **foundation phase** — the brief's Weeks 1–2:

- Repo, Drizzle schema + migrations, auth (phone OTP + Google), base layout
- Film/person/producer pages, search, where-to-watch links, `/out` click tracking, SEO scaffolding

Later phases (ingestion, social, trending, producer studio, campaigns, moderation) each get their own spec → plan → build cycle against this foundation.

**Engagement decisions already made:**

- **Local-first development.** No external services are provisioned yet. Every vendor sits behind an adapter interface with a dev implementation; provisioning Neon/Termii/R2/Vercel later is configuration, not code.
- **Repo lives at `~/projects/orban-forest`** (WSL ext4 — `/mnt/e` is too slow for Node tooling). The original brief is copied into `docs/`.
- **Custom auth** (sessions table + hand-rolled OTP + Arctic for Google), not Auth.js or a hosted provider.
- **Local Postgres 16 via apt**, not Docker (unavailable in this WSL distro) or PGlite (parity risk).
- **Visual direction established by Claude** (§6 below), judged by the founder on real screens during build.

### Acceptance criteria (from the brief)

1. Sign up by phone (dev OTP), session persists across restarts.
2. A film page renders ranks-ready: valid schema.org `Movie` JSON-LD, correct title template, working where-to-watch links that log through `/out`, and an OG card that meets WhatsApp's preview constraints (1200×630, < 300KB, correct OG tags — verified locally; a live WhatsApp preview test happens at first deploy).
3. Search returns films by title (typo-tolerant).
4. CI enforces typecheck, lint (incl. module boundaries), tests, and the 150KB first-load JS budget.

## 1. Architecture and module boundaries

Single Next.js 15 app (App Router, TypeScript, pnpm), vertical feature modules:

```
src/
  db/            schema.ts (all tables from brief §2), migrations/, seed/
  lib/           adapters (sms/, storage/), config.ts, errors.ts, rate-limit.ts
  modules/
    auth/        OTP flow, sessions, Google OAuth (Arctic), guards
    catalog/     films, people, credits, film_links queries
    events/      logEvent() — the ONLY writer to the events table
    search/      Postgres FTS + trigram
    seo/         JSON-LD builders, metadata helpers, sitemap generation
    trending/    (empty placeholder — Week 5)
    feed/        (empty placeholder — Week 5)
    campaigns/   (empty placeholder — Week 7)
  app/           routes — thin; all logic lives in modules/
```

**Structural rules, enforced by ESLint `no-restricted-imports` in CI from day one:**

1. `app/` never imports from `db/` directly — always through a module.
2. `modules/trending` and `modules/feed` must not import from `modules/campaigns` (the brief's trust invariant §0). The placeholder directories exist now so the lint rule is active before the modules have code.

**Adapter pattern for vendors:**

- `SmsProvider` — `TermiiSms` (real, unused until keyed) and `DevSms` (logs OTP to console; fixed code `000000` accepted in dev). Selected by `SMS_PROVIDER` env var.
- `ImageStorage` — `R2Storage` (real, unused until keyed) and `LocalStorage` (`./public/uploads`). Selected by `STORAGE_PROVIDER` env var.

## 2. Data layer

**Complete schema in migration 0001** — all tables from brief §2 (users, films, people, film_credits, film_links, producers, producer_films, ratings, reviews, lists, list_items, follows, watch_intent, four_favorites, events, wallets, wallet_txns, campaigns, ingest_candidates, edits, flags, trending_scores), transcribed to Drizzle, even though this phase only actively uses a subset. Later phases build features, not migrations, and cross-table FKs exist from the start.

**Additions beyond the brief's DDL:**

- `sessions` (id, user_id FK, token_hash, expires_at, created_at) — auth.
- `otp_codes` (id, phone, code_hash, expires_at, attempts, consumed_at, created_at) — auth.
- `youtube_channels` (id, channel_id, name, added_at) — named in brief §5/P2 but missing from its DDL; included now so Week 3 needs no migration.
- Films FTS: generated `tsvector` column over title + synopsis with GIN index, plus the `pg_trgm` extension and a trigram index on title for typo tolerance.

**Local database:** Postgres 16 installed via apt in WSL. `DATABASE_URL` in `.env.local`. Separate `orban_forest_test` database for integration tests (schema pushed per run). Swapping to Neon later is a connection-string change.

**Seed data:** `pnpm db:seed` loads ~50 realistic Nollywood films (real public-knowledge titles and people, varied genres/languages/years), with a deliberate mix of link states — YouTube links, Netflix links, cinema-only, and some with **zero** film_links (the brief treats link absence as demand signal). Includes a handful of people and producers so `/person` and `/producer` pages render.

## 3. Auth

Custom, phone-OTP-first (brief §9), no auth framework.

**Sessions:**

- `sessions` table; cookie stores an opaque random token, SHA-256 hash stored at rest.
- Cookie: `httpOnly`, `Secure` (prod), `SameSite=Lax`, 30-day sliding expiry (refreshed when < 15 days remain).
- Middleware handles only session refresh; authorization lives in guards.

**Phone OTP flow:**

- `POST /api/auth/otp/send` — normalize to E.164 (Nigerian default country code), rate-limit (3/phone/hour, 10/IP/hour), generate 6-digit code, hash at rest, 10-minute expiry, send via `SmsProvider`.
- `POST /api/auth/otp/verify` — max 5 attempts per code, then consumed. On success: find-or-create user by phone, set `phone_verified_at`, bump `trust_weight` 0.2 → 0.6 (brief §9 schedule), create session.
- New users complete a username step after first verify (auto-suggested, editable; unique, lowercase, 3–20 chars `[a-z0-9_]`). The same step applies to first-time Google-OAuth signups.

**Google OAuth (built, dormant):**

- Arctic library; `/api/auth/google` → consent → `/api/auth/google/callback`.
- Links to an existing user by verified email, else creates one (phone stays null — "OAuth-only" per the brief's users DDL).
- Hidden behind a config flag until `GOOGLE_CLIENT_ID/SECRET` exist; code is complete and unit-tested against a mocked token endpoint.

**Guards (server-side helpers):** `getUser()` (nullable), `requireUser()`, `requireVerifiedPhone()` (the future gate for ratings/reviews), `requireRole('producer' | 'admin')`. Used by route handlers and server components.

**Rate limiting:** Postgres-backed fixed-window counter (`lib/rate-limit.ts`) keyed by (scope, identifier, window). No Redis in the $50 envelope; a table is fine at this scale.

## 4. Viewer pages and SEO

**Routes this phase:**

| Route | Content |
|---|---|
| `/` | Minimal home: recent-releases poster grid ordered by `release_date` (full home with trending/feed rails is Week 5) |
| `/film/[slug]` | Flagship page: poster, title/year/runtime/languages/genres/synopsis, credits grouped by role, where-to-watch buttons (via `/out`), NFVCB ref if present; empty-state slots where ratings/reviews mount in Week 4 |
| `/person/[slug]` | Photo, name, filmography grouped by credit |
| `/producer/[slug]` | Logo, bio, socials, claimed films (approved `producer_films` only) |
| `/search` | Server-rendered results; FTS + trigram fallback |
| Auth pages | Phone entry → OTP → username step; login/logout |

**SEO scaffolding (brief §4):**

- ISR with tag-based revalidation: `revalidateTag('film:{slug}')` on any film edit.
- `modules/seo` JSON-LD builders: `Movie` (film pages), `Person` (people). Unit-tested for schema.org validity.
- Title template: `"{Title} ({Year}) — where to watch, reviews, cast"`.
- Partitioned sitemap: `/sitemap.xml` index → film and people partitions; regenerated by the (future) daily cron, and on-demand on publish. This phase ships the generator + route.
- `robots.txt` allowing all, pointing at the sitemap.
- OG cards via `@vercel/og`: `/og/film/[slug]` — poster, title, year, where-to-watch platform badges. 1200×630, tested against WhatsApp's preview requirements (< 300KB, absolute URLs, `og:image:width/height` tags).

**Performance budget (brief §3a), enforced in CI now:**

- First-load JS < 150KB gzipped on viewer routes (checked in CI against build output).
- Posters through `next/image` with srcset; feed-size thumbnails ≤ 40KB.
- Zero third-party scripts on viewer surfaces.
- Single self-hosted variable font, subset.

## 5. Events and `/out` tracking

- `modules/events` exposes `logEvent(type, ctx)`; it is the only code that inserts into `events` (lint-enforced: no other module imports the events table schema).
- Anonymous visitors get an `anon_id` cookie (random ID, 1-year, no PII) set lazily on first event-producing action.
- Events emitted this phase: `film_view` (server-side on film page render), `outbound_click`.
- `/out/[linkId]`: look up `film_links` row → `logEvent('outbound_click', {filmId, surface})` → 302 to the destination URL. Logging is fire-and-forget: a logging failure never blocks the redirect. Unknown or malformed linkId → 404.

## 6. Visual direction

Dark cinematic, poster-forward, mobile-first:

- **Ground:** near-black warm charcoal (not pure black), with poster art acting as the interface's primary color. OLED-friendly and data-light.
- **Accent:** one saturated forest-green family for interactive elements (nod to the name).
- **Type:** single self-hosted variable font with strong display weights; subset aggressively (performance decision as much as aesthetic).
- **Layout:** dense poster grids like a video shelf, not a text database. Designed at 360px width first; desktop is the adaptation (brief §3a).
- Implemented as Tailwind design tokens, so direction changes are cheap. The founder judges on real screens during build; this is a starting direction, not a locked brand.

## 7. Error handling

- Typed `AppError` hierarchy in `lib/errors.ts` (`NotFoundError`, `ValidationError`, `RateLimitError`, `AuthError`).
- API route handlers return consistent JSON: `{ error: { code, message } }` with matching HTTP status.
- Pages get `not-found.tsx` and `error.tsx` boundaries with designed 404/500 states.
- Zod validation at every API boundary; validation failures are 400s with field detail.

## 8. Testing and CI

- **Unit (Vitest):** OTP lifecycle (generation, expiry, attempt cap, rate limits), session create/refresh/expire, JSON-LD output validity, slug generation, E.164 normalization.
- **Integration (Vitest, real local Postgres test DB):** auth end-to-end (send → verify → session → guard), `/out` logs then redirects, search returns expected films, seed script idempotency.
- **CI (GitHub Actions):** typecheck → ESLint (incl. both module-boundary rules) → tests (Postgres service container) → build → bundle-size budget check.
- TDD throughout implementation per the superpowers process.

## Out of scope for this phase

Ingestion pipelines and admin queue (Week 3), ratings/reviews/lists/follows/watchlist/profiles (Week 4), trending + PWA + push (Week 5), producer studio (Week 6), wallet/Paystack/campaigns (Week 7), moderation/trust-weight progression/launch checklist (Week 8). Vercel deployment happens when accounts are provisioned; nothing in this phase blocks on it.
