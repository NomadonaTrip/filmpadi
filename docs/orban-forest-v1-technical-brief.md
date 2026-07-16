# Orban Forest — v1 Technical Brief (Attention Platform)

**Version:** 1.0 · July 2026 · Companion to Strategic Brief v6.0
**Supersedes:** the v5-era Markdown technical brief (Android/Widevine/FastAPI), which is archived against v2.
**Builder:** solo founder + Claude Code.

---

## 0. What v1 is (and is not)

A web-based Nollywood discovery platform. Viewers get the complete, current picture of Nollywood (recency promise), trend signals, and social discovery — free. Producers get verified pages, an attribution dashboard, and paid promoted placement. The platform hosts **metadata only** — never film files.

**Not in v1:** DRM, video hosting/streaming, offline downloads, payments *to* producers, viewer-side payments of any kind, ML recommendations, native apps.

### Trust invariants (engineering-level, non-negotiable)

These implement Strategic Brief v6.0 §6.1 and §5.4. Every PR touching feeds, ranking, or campaigns is checked against them:

- [ ] Every promoted impression renders a visible "Promoted" label. No unlabeled paid surface exists anywhere.
- [ ] Campaign spend has **no code path** into: ratings aggregates, review ordering, list rankings, follower feeds, or trending computation. Enforce by module boundary — the trending and organic-feed modules must not import campaign data.
- [ ] Trending is computed from organic signals only, velocity-based (see §7).
- [ ] Producer analytics always split organic vs. paid attention.
- [ ] Trend surfacing below the traffic-maturity threshold requires editorial approval before display.

---

## 1. Stack

| Layer | Choice | Notes |
|---|---|---|
| App | Next.js 15 (App Router, TypeScript) on Vercel | SSR/ISR for SEO; single app for viewer, producer, and admin surfaces |
| DB | Postgres (Neon) | Serverless-friendly; branchable for dev |
| ORM | Drizzle | Schema-as-code, SQL-first |
| Auth | Phone OTP via Termii (primary) + Google OAuth (secondary) | Phone-first matches the Nigerian market; Termii already vendored |
| Images | Cloudflare R2 + Cloudflare Images | Posters, avatars; stays in the existing Cloudflare account |
| Payments (producer wallet) | Paystack | Naira, cards + bank transfer + USSD. Money flows one direction: producer → platform |
| Background jobs | Vercel Cron → route handlers; Inngest if jobs outgrow cron | Ingestion, trending recompute, digests |
| OG/share cards | @vercel/og | Dynamic per film/review/list; WhatsApp-preview-optimized |
| Analytics events | Postgres `events` table (first-party) | The taste graph's raw feed; no third-party analytics on viewer data |

Target infra cost: within the existing ~$50/month envelope (Neon free/launch tier, Vercel Pro, R2 pennies).

---

## 2. Data model (DDL)

```sql
-- Identity ---------------------------------------------------------------
CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  phone         text UNIQUE,             -- E.164; null if OAuth-only
  email         text UNIQUE,
  role          text NOT NULL DEFAULT 'viewer',  -- viewer | producer | admin
  username      text UNIQUE NOT NULL,
  display_name  text,
  avatar_url    text,
  bio           text,
  created_at    timestamptz NOT NULL DEFAULT now(),
  -- anti-gaming inputs
  phone_verified_at timestamptz,
  trust_weight  real NOT NULL DEFAULT 0.2   -- grows with account age + verified phone
);

-- Catalog ----------------------------------------------------------------
CREATE TABLE films (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  slug          text UNIQUE NOT NULL,
  title         text NOT NULL,
  release_date  date,                    -- drives the recency promise
  year          int,
  synopsis      text,
  poster_url    text,
  runtime_min   int,
  languages     text[] NOT NULL DEFAULT '{}',   -- ['yoruba','english',...]
  genres        text[] NOT NULL DEFAULT '{}',
  nfvcb_ref     text,                    -- Censors Board classification ref
  status        text NOT NULL DEFAULT 'published', -- draft|published|flagged|removed
  source        text NOT NULL,           -- nfvcb|youtube|producer|community|editorial
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX films_release_idx ON films (release_date DESC) WHERE status = 'published';

CREATE TABLE people (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  slug text UNIQUE NOT NULL,
  name text NOT NULL,
  photo_url text
);

CREATE TABLE film_credits (
  film_id   uuid REFERENCES films(id) ON DELETE CASCADE,
  person_id uuid REFERENCES people(id) ON DELETE CASCADE,
  credit    text NOT NULL,               -- director|producer|cast|writer
  billing   int,
  PRIMARY KEY (film_id, person_id, credit)
);

-- Where-to-watch (the conversion surface) ---------------------------------
CREATE TABLE film_links (
  id        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  film_id   uuid NOT NULL REFERENCES films(id) ON DELETE CASCADE,
  platform  text NOT NULL,               -- youtube|netflix|prime|cinema|other
  url       text NOT NULL,
  label     text,                        -- e.g. cinema chain name
  verified  boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now()
);
-- "No link available" is itself demand signal (v2 evidence): a published
-- film with zero film_links rows is queryable on purpose.

-- Producers ----------------------------------------------------------------
CREATE TABLE producers (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  slug        text UNIQUE NOT NULL,
  name        text NOT NULL,
  owner_user  uuid REFERENCES users(id),
  verified_at timestamptz,
  bio         text, logo_url text,
  socials     jsonb NOT NULL DEFAULT '{}'
);

CREATE TABLE producer_films (
  producer_id uuid REFERENCES producers(id) ON DELETE CASCADE,
  film_id     uuid REFERENCES films(id) ON DELETE CASCADE,
  claim_state text NOT NULL DEFAULT 'pending',  -- pending|approved|rejected
  PRIMARY KEY (producer_id, film_id)
);

-- Social -------------------------------------------------------------------
CREATE TABLE ratings (
  user_id uuid REFERENCES users(id) ON DELETE CASCADE,
  film_id uuid REFERENCES films(id) ON DELETE CASCADE,
  stars   real NOT NULL CHECK (stars BETWEEN 0.5 AND 5),
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, film_id)
);

CREATE TABLE reviews (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  film_id uuid NOT NULL REFERENCES films(id) ON DELETE CASCADE,
  body text NOT NULL,                    -- tweet-length encouraged, 2000 char cap
  spoiler boolean NOT NULL DEFAULT false,
  status text NOT NULL DEFAULT 'live',   -- live|flagged|removed
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE lists (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  slug text NOT NULL, title text NOT NULL, description text,
  is_ranked boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (user_id, slug)
);
CREATE TABLE list_items (
  list_id uuid REFERENCES lists(id) ON DELETE CASCADE,
  film_id uuid REFERENCES films(id) ON DELETE CASCADE,
  position int NOT NULL, note text,
  PRIMARY KEY (list_id, film_id)
);

CREATE TABLE follows (
  follower uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  -- exactly one of the below
  followed_user uuid REFERENCES users(id) ON DELETE CASCADE,
  followed_producer uuid REFERENCES producers(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  CHECK ((followed_user IS NULL) <> (followed_producer IS NULL))
);

CREATE TABLE watch_intent (
  user_id uuid REFERENCES users(id) ON DELETE CASCADE,
  film_id uuid REFERENCES films(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, film_id)
);

CREATE TABLE four_favorites (
  user_id uuid REFERENCES users(id) ON DELETE CASCADE,
  film_id uuid REFERENCES films(id) ON DELETE CASCADE,
  position int NOT NULL CHECK (position BETWEEN 1 AND 4),
  PRIMARY KEY (user_id, position)
);

-- Events + attribution (taste graph raw feed) -------------------------------
CREATE TABLE events (
  id bigserial PRIMARY KEY,
  user_id uuid,                          -- null for anonymous
  anon_id text,                          -- cookie id for anonymous
  type text NOT NULL,   -- film_view|outbound_click|watch_intent_add|rating|review|list_add|follow|share
  film_id uuid, producer_id uuid,
  surface text,         -- home|new_releases|trending|feed|film_page|list|search|promoted
  campaign_id uuid,     -- ONLY set when surface='promoted'
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX events_film_time_idx ON events (film_id, type, created_at DESC);

-- Outbound click tracking: /out/{film_link_id} logs event then 302s.

-- Campaigns (one-directional money) -----------------------------------------
CREATE TABLE wallets (
  producer_id uuid PRIMARY KEY REFERENCES producers(id),
  balance_kobo bigint NOT NULL DEFAULT 0 CHECK (balance_kobo >= 0)
);
CREATE TABLE wallet_txns (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  producer_id uuid NOT NULL REFERENCES producers(id),
  amount_kobo bigint NOT NULL,           -- +topup (Paystack), -campaign spend
  kind text NOT NULL,                    -- topup|spend|refund
  paystack_ref text,
  campaign_id uuid,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE campaigns (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  producer_id uuid NOT NULL REFERENCES producers(id),
  film_id uuid NOT NULL REFERENCES films(id),
  status text NOT NULL DEFAULT 'draft',  -- draft|active|paused|exhausted|ended
  daily_budget_kobo bigint NOT NULL,
  total_budget_kobo bigint NOT NULL,
  spent_kobo bigint NOT NULL DEFAULT 0,
  surfaces text[] NOT NULL DEFAULT '{home,genre_feed}',  -- NEVER 'trending'
  targeting jsonb NOT NULL DEFAULT '{}', -- v1: genres/languages; taste segments later
  starts_at timestamptz, ends_at timestamptz
);

-- Ingestion + moderation -----------------------------------------------------
CREATE TABLE ingest_candidates (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source text NOT NULL,                  -- nfvcb|youtube|producer|community
  raw jsonb NOT NULL,
  matched_film uuid REFERENCES films(id),
  state text NOT NULL DEFAULT 'new',     -- new|merged|created|rejected
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE edits (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES users(id),
  film_id uuid REFERENCES films(id),
  patch jsonb NOT NULL,
  state text NOT NULL DEFAULT 'pending', -- pending|approved|rejected|auto
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE flags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid, target_type text NOT NULL, target_id uuid NOT NULL,
  reason text, state text NOT NULL DEFAULT 'open',
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE trending_scores (
  film_id uuid PRIMARY KEY REFERENCES films(id),
  score real NOT NULL,
  window_signals jsonb NOT NULL,         -- debugging: per-signal velocities
  editorial_state text NOT NULL DEFAULT 'auto',  -- auto|approved|held
  computed_at timestamptz NOT NULL
);
```

---

## 3. Route map (App Router)

**Viewer (public, SSR/ISR):**
`/` home (new releases rail + trending rail + activity feed) · `/new` full recent-releases surface, filterable by genre/language/week · `/trending` · `/film/[slug]` · `/person/[slug]` · `/producer/[slug]` · `/list/[user]/[slug]` · `/u/[username]` profile · `/genre/[g]`, `/language/[l]` daily feeds · `/search`

**Viewer (authed):** watchlist, diary, settings, review/rating/list composers (modal or route).

**Producer (authed, role=producer):**
`/studio` dashboard (organic vs paid split) · `/studio/films` claims · `/studio/campaigns` + `/studio/campaigns/new` · `/studio/wallet` (Paystack top-up)

**Admin:** `/admin/ingest` candidate queue · `/admin/edits` · `/admin/flags` · `/admin/trending` (approve/hold below maturity threshold) · `/admin/producers` verification queue

**System routes:**
`/out/[linkId]` — log outbound_click event, 302 redirect (the conversion event)
`/og/film/[slug]`, `/og/review/[id]`, `/og/list/[id]` — @vercel/og share cards
`/sitemap.xml`, `/robots.txt` — sitemap regenerated on publish
`/api/cron/*` — see §6

**API route handlers (JSON):** ~22 endpoints following the route map (films CRUD-lite, ratings, reviews, lists, follows, watch-intent, search, feed, producer claims, campaigns, wallet webhook `paystack/webhook` with signature verification, admin actions). Keep REST-ish, colocated with features.

---

## 3a. Mobile strategy: mobile-first web + PWA, no native app in v1

Most of the audience is on mobile — but v1's acquisition loops (WhatsApp share cards, Google search) live in the browser, and an app-store install wall would break both. Low-end Android storage pressure and data costs make installs a tax; the web is the front door. Native Android returns at v2 with DRM/offline downloads (its original technical justification), or at v1.x only if retention data shows web push underperforming.

**PWA requirements:**
- Web app manifest + service worker: installable from Chrome (dominant browser), offline app shell, cached poster thumbnails.
- Web push notifications (Android Chrome): new-release alerts for followed producers, watchlist availability, weekly digest. Prompt only after an engagement action, never on first visit.
- All layouts designed mobile-first; desktop is the adaptation.

**Performance budget (enforced in CI, tested on throttled 3G / low-end Android profile):**
- First load JS < 150KB gzipped on viewer routes; film page LCP < 2.5s on Slow 4G.
- Images: Cloudflare Images with aggressive srcset; posters served ≤ 40KB at feed size.
- No third-party scripts on viewer surfaces.

## 4. SEO (the primary acquisition channel)

- ISR for all film/person/producer/list pages; revalidate on edit.
- JSON-LD `Movie` schema on film pages; `Person` on people; `ItemList` on lists.
- Film page `<title>`: `"{Title} ({Year}) — where to watch, reviews, cast"` — targets the "watch {title}" query space where no structured result currently exists.
- Sitemap partitioned (films/people/lists), pinged on publish.
- OG cards: film poster + rating + "where to watch" badges; review cards: quote + stars + poster. Test rendering in WhatsApp specifically — it is the primary share surface.

---

## 5. Ingestion pipelines

All pipelines write to `ingest_candidates`, never directly to `films`. A matcher (title + year fuzzy match, normalized) merges into existing films or creates new ones; low-confidence matches go to `/admin/ingest`.

**P1 — NFVCB (recency SLA backbone):** scheduled scrape/fetch of new classification records → candidates. If no machine-readable source exists, scrape the published listings; build resilient parsing with snapshot tests. Frequency: daily.

**P2 — YouTube:** maintain a `youtube_channels` seed table of known Nollywood channels. Poll uploads via YouTube Data API (or RSS to spare quota) → new full-film uploads become candidates with auto-attached `film_links` (platform=youtube). Frequency: every 6h. Heuristics: duration > 45min, title patterns; everything else to admin queue.

**P3 — Producer self-service:** verified producers add/edit their films directly (state=auto with audit row in `edits`); unverified claims queue for review.

**P4 — Community edits:** wiki-style patches into `edits`, moderated. Users above a trust_weight threshold get auto-approve for minor fields.

**Seeding (one-time, Claude Code task):** backfill from NFVCB archives + YouTube channel enumeration to launch with a few-thousand-film catalog. Budget ~1–2 weeks of pipeline runs and manual QA of the top 500 titles.

---

## 6. Cron schedule

| Job | Freq | Work |
|---|---|---|
| `ingest-nfvcb` | daily | P1 |
| `ingest-youtube` | 6h | P2 |
| `trending-recompute` | hourly | §7 |
| `campaign-budgeter` | hourly | pause campaigns at daily/total budget; decrement wallets atomically |
| `digest` | weekly | email/WhatsApp-able "this week in Nollywood" (deferrable) |
| `sitemap-refresh` | daily | regenerate partitions |

---

## 7. Trending algorithm (v6.0 §5.4)

Velocity over volume; organic signals only; no ML.

```
signals per film, trailing windows:
  W = watch_intent adds, R = reviews+ratings, V = film_page views, C = outbound clicks

weighted rate now  = Σ wᵢ · countᵢ(last 24h) · trust_weight-adjusted, deduped per user
baseline rate      = Σ wᵢ · countᵢ(prior 7d) / 7  (+ smoothing constant k to damp tiny bases)

score = rate_now / (baseline + k)      -- acceleration relative to the film's own history
```

- Suggested weights: C=4, W=3, R=2, V=1 (clicks are the strongest intent).
- **Dedup:** one contribution per user per signal per film per window.
- **Trust weighting:** multiply each user's contribution by `users.trust_weight` (0.2 at signup → 1.0 with verified phone + 30 days age). Blunts coordinated new-account gaming.
- **Maturity gate:** if unique contributing users < N (configurable, start N=25), `editorial_state='held'` — appears in `/admin/trending` for approval, not on the public surface.
- **Isolation:** the recompute job reads `events` WHERE `campaign_id IS NULL`. Promoted impressions and their downstream clicks never enter trending. Enforced in code review via the trust-invariant checklist.

Display: badges on the new-releases surface ("Trending" / "Breaking out"), plus the `/trending` rail. Store `window_signals` for explainability and gaming forensics.

---

## 8. Campaigns and attribution

- **Wallet:** prepaid kobo balance, Paystack top-up via webhook (verify `x-paystack-signature`). No credit; campaigns pause at zero.
- **Serving:** promoted slots are fixed positions in home and genre feeds (e.g., position 3 and 9), clearly labeled, frequency-capped per user per day. v1 pricing: flat CPM equivalent charged per impression event; keep the rate card in config, not code.
- **Attribution dashboard:** per film, per period: page views, follows, watch-intent adds, outbound clicks — **split organic vs paid** (paid = events with campaign_id). Never promise sales attribution (v6.0 §6, honest constraint).

---

## 9. Auth and anti-gaming

- Phone OTP (Termii) primary; unverified accounts can browse and watchlist but ratings/reviews require verified phone. This is both spam control and trust_weight bootstrap.
- Rate limits on write endpoints (per user + per IP).
- `trust_weight` schedule: 0.2 signup → 0.6 verified phone → 1.0 at 30 days with activity. Admin can zero a user (shadow-weight).

---

## 10. Build sequence (8 weeks)

| Wk | Deliverable | Acceptance |
|---|---|---|
| 1 | Repo, Drizzle schema + migrations, auth (Termii OTP + Google), base layout | Sign up by phone, session persists |
| 2 | Film/person/producer pages, search, where-to-watch links, `/out` tracking, SEO scaffolding (ISR, JSON-LD, sitemap) | A film page ranks-ready: valid schema.org, OG card renders in WhatsApp |
| 3 | Ingestion: candidates + matcher + admin queue; NFVCB + YouTube pipelines; begin catalog seeding | Pipelines produce deduped candidates; 500+ films live |
| 4 | Social: ratings, reviews, lists, follows, watchlist, four favorites, profiles, activity feed | Full loop: rate → review → share card → follow |
| 5 | New-releases surface + genre/language daily feeds + trending (compute, badges, admin gate) + PWA (manifest, service worker, web push) | Trending recompute hourly; held-state works below maturity gate; installable on Android Chrome, push received |
| 6 | Producer verification, claims, studio dashboard with organic analytics | Producer sees per-film views/follows/intent/clicks |
| 7 | Wallet + Paystack + campaigns (serving, labeling, budget enforcement) + paid/organic split in dashboard | Top-up → campaign live → labeled impressions → spend decrements → auto-pause |
| 8 | Moderation (edits/flags), rate limits, trust weights, share-card polish, launch checklist incl. trust-invariant audit | Invariant checklist passes; seed catalog QA'd; soft launch |

**Kill-criterion instrumentation (build in week 7, not later):** repeat-campaign rate per producer must be a first-class metric from the first campaign sold.

---

## 11. Environment

```
DATABASE_URL=            # Neon
TERMII_API_KEY=
GOOGLE_CLIENT_ID= / GOOGLE_CLIENT_SECRET=
PAYSTACK_SECRET_KEY= / PAYSTACK_PUBLIC_KEY=
R2_ACCOUNT_ID= / R2_ACCESS_KEY= / R2_SECRET= / R2_BUCKET=
YOUTUBE_API_KEY=
CRON_SECRET=             # protect /api/cron/*
TRENDING_MATURITY_N=25
```
