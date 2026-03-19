# Codebase Structure

**Analysis Date:** 2026-03-19

## Directory Layout

```
/Users/troyhardister/chairlyos/mostlypostly/
├── mostlypostly-app/                # Main Node.js application
│   ├── server.js                    # Express app entry point — middleware, routes, scheduler
│   ├── db.js                        # SQLite connection singleton, schema/migrations init
│   ├── package.json                 # Dependencies
│   ├── schema.sql                   # Base schema (CREATE TABLE IF NOT EXISTS)
│   ├── .render.yaml                 # Render.com deployment config
│   ├── migrations/                  # DB migrations (001–043)
│   ├── public/                      # Static assets
│   │   ├── admin.js                 # Client-side modal system for admin pages
│   │   ├── logo/                    # Brand assets (logo-trimmed.png, logo-mark.png)
│   │   └── uploads/                 # User-uploaded files (temporary until app init)
│   ├── src/
│   │   ├── env.js                   # Load .env, validate required vars
│   │   ├── openai.js                # OpenAI Vision caption generation entry point
│   │   ├── scheduler.js             # Background post scheduler — polling, publishing, retries
│   │   ├── core/                    # Business logic modules
│   │   ├── db/                      # DB-specific helpers (seedStaging)
│   │   ├── middleware/              # Express middleware
│   │   ├── publishers/              # External API clients (Facebook, Instagram, GMB)
│   │   ├── routes/                  # Request handlers (auth, dashboard, webhooks, admin)
│   │   └── ui/                      # Server-rendered HTML components (pageShell.js)
│   └── tests/                       # Unit/integration tests (Jest)
└── mostlypostly-site/               # Static marketing site
    ├── index.html                   # Main landing page
    ├── about.html                   # Founder story
    ├── founders.html                # Founder offer landing
    ├── watch.html                   # Video demo page
    ├── vendors.html                 # Vendor integrations page
    ├── kb.html                      # Knowledge base hub
    ├── kb/                          # Knowledge base articles (9 sub-pages)
    ├── legal/                       # Terms, privacy, SMS consent
    └── public/                      # Fonts, images, CSS overrides
```

## Directory Purposes

**`mostlypostly-app/`:**
- Purpose: Main application serving dashboard, webhooks, auth, posting, integrations
- Contains: Express.js server, SQLite database, business logic, publishers
- Key files: `server.js` (entry), `db.js` (DB init), `scheduler.js` (background jobs)

**`src/`:**
- Purpose: Application source code (organized by layer)
- Contains: Core logic, routes, middleware, UI, publishers
- Key files: See subdir breakdown below

**`src/core/`:**
- Purpose: Implement domain features (40+ modules)
- Contains:
  - Message processing: `messageRouter.js`, `joinManager.js`, `availabilityRequest.js`
  - Caption/image: `composeFinalCaption.js`, `buildAvailabilityImage.js`, `buildPromotionImage.js`, `celebrationImageGen.js`
  - Integrations: `zenoti.js`, `zenotiSync.js`, `zenotiAvailability.js`, `fetchGmbInsights.js`, `googleTokenRefresh.js`
  - Scheduling: `celebrationScheduler.js`, `vendorScheduler.js`
  - Tracking: `trackingUrl.js`, `utm.js`, `postErrorTranslator.js`
  - Utilities: `storage.js`, `salonLookup.js`, `email.js`, `encryption.js`, `gamification.js`

**`src/routes/`:**
- Purpose: HTTP request handlers
- Contains (22 route files):
  - Auth: `managerAuth.js`, `googleAuth.js`, `facebookAuth.js`, `mfa.js`
  - Dashboard: `dashboard.js`, `posts.js`, `postQueue.js` (drag-and-drop scheduler)
  - Management: `manager.js` (approval page), `stylistManager.js` (team page), `admin.js` (salon settings), `managerProfile.js`
  - Webhooks: `twilio.js`, `telegram.js`, `tracking.js` (URL redirect/logging)
  - Integrations: `integrations.js` (Zenoti/GMB UI), `onboarding.js` (signup flow), `facebookAuth.js`, `googleAuth.js`
  - Analytics: `analytics.js`, `analyticsScheduler.js`, `leaderboard.js`, `teamPerformance.js`
  - Admin: `vendorAdmin.js` (`/internal/vendors` console), `vendorFeeds.js` (salon-facing vendor UI)
  - Special: `stylistPortal.js` (token-auth portal), `locations.js` (multi-location switcher), `help.js`, `schedulerConfig.js`, `billing.js`, `onboardingGuard.js`, `internal.js`

**`src/publishers/`:**
- Purpose: Publish to external platforms
- Contains:
  - `facebook.js` — Graph API photo/video posts
  - `instagram.js` — Graph API media + carousel + story posts
  - `googleBusiness.js` — GMB API "What's New" + "Offers"

**`src/middleware/`:**
- Purpose: Express middleware functions
- Contains:
  - `csrf.js` — CSRF token generation and verification (session-based)
  - `tenantFromLink.js` — Multi-tenant resolution (future expansion)

**`src/ui/`:**
- Purpose: Server-rendered HTML components
- Contains:
  - `pageShell.js` — Shared HTML shell (header, sidebar nav, mobile menu, Tailwind + font CDN)

**`migrations/`:**
- Purpose: Database schema evolution (numbered 001–043, runs sequentially)
- Contains: One file per migration; idempotent with `migration_runner.js`
- Examples: `015_salon_groups.js` (multi-location), `039_gmb.js` (Google Business), `043_utm_tracking.js`

**`public/`:**
- Purpose: Static assets served at `/public/`
- Contains:
  - `admin.js` — Client-side modal system (for Admin page form handling)
  - `logo/` — Brand assets (logo-trimmed.png for web, logo-mark.png for mobile menu)
  - `uploads/` — User-uploaded stock photos, logos (symlink or real directory)

**`mostlypostly-site/`:**
- Purpose: Static marketing website (separate from app)
- Contains: 8 main HTML pages + 9 KB articles + 3 legal pages
- Key files:
  - `index.html` — Main landing page (pricing, features, CTAs)
  - `watch.html` — Video demo page (stylist reel + manager tour)
  - `vendors.html` — Vendor brand integrations page
  - `kb.html` — Knowledge base hub (links to 9 articles)

## Key File Locations

**Entry Points:**
- `mostlypostly-app/server.js`: Web server initialization, route registration, scheduler startup
- `mostlypostly-app/db.js`: Database connection, schema/migrations execution
- `mostlypostly-app/src/scheduler.js`: Background post publishing loop

**Configuration:**
- `.env` — Secrets (Twilio, OpenAI, Stripe, Google, etc.)
- `src/env.js` — Environment validation on startup
- `package.json` — Dependencies (express, better-sqlite3, twilio, openai, luxon, etc.)
- `.render.yaml` — Render.com deploy config (build, start, cron jobs)

**Core Business Logic:**
- `src/core/messageRouter.js` — Main message processor (stylist SMS/Telegram handler)
- `src/core/salonLookup.js` — Stylist/salon identification from phone/chat_id
- `src/core/storage.js` — Post/draft persistence helpers
- `src/scheduler.js` — Post publishing scheduler

**Image Generation:**
- `src/core/buildAvailabilityImage.js` — 1080×1920 availability posts
- `src/core/buildPromotionImage.js` — 1080×1920 promotion posts
- `src/core/celebrationImageGen.js` — Dual-format (feed + story) celebration images

**Integrations:**
- `src/core/zenoti.js` — Zenoti API client (working hours, appointments, services)
- `src/core/zenotiSync.js` — Zenoti availability pooling and post generation
- `src/core/fetchGmbInsights.js` — GMB insights sync
- `src/core/googleTokenRefresh.js` — GMB token refresh

**Publishers:**
- `src/publishers/facebook.js` — Facebook Graph API
- `src/publishers/instagram.js` — Instagram Graph API
- `src/publishers/googleBusiness.js` — GMB API

**Dashboard & Admin:**
- `src/routes/manager.js` — Manager approval page and one-off publish
- `src/routes/admin.js` — Admin panel (salon settings, team, stock photos, hashtags, branding)
- `src/routes/analytics.js` — Analytics dashboard (insights, syncing, trending)
- `src/routes/dashboard.js` — Post database view

**Testing:**
- `tests/` — Jest test files (structure mirrors `src/`)

## Naming Conventions

**Files:**
- Routes: camelCase (e.g., `managerAuth.js`, `stylistManager.js`)
- Core modules: camelCase with domain prefix (e.g., `buildAvailabilityImage.js`, `zenotiSync.js`)
- Migrations: `NNN_description.js` (e.g., `015_salon_groups.js`, `039_gmb.js`)
- Exports: default export preferred; named exports for utilities

**Directories:**
- Lowercase, plural for collections (e.g., `migrations/`, `publishers/`, `routes/`)
- Grouping by layer/concern (e.g., `core/`, `publishers/`, `middleware/`)

**Database:**
- Tables: lowercase plural (e.g., `posts`, `stylists`, `salons`, `managers`)
- Columns: lowercase, underscores for compound names (e.g., `salon_id`, `created_at`, `facebook_page_id`)
- Migrations: numbered and ordered (001, 002, ..., 043)

**Routes/URLs:**
- Kebab-case in URLs (e.g., `/manager/forgot-password`, `/auth/google/callback`)
- Nested under domain (e.g., `/manager/*` for manager dashboard, `/auth/*` for OAuth)

**Functions:**
- camelCase: `handleIncomingMessage()`, `buildTrackingToken()`, `publishToFacebook()`
- Async functions: prefix with `async`, use promises not callbacks

**Variables:**
- camelCase: `draftId`, `salonId`, `stylistName`
- Constants: UPPER_SNAKE_CASE (e.g., `FORCE_POST_NOW`, `FEED_TYPES`)
- Booleans: prefix with `is`, `has`, `can` (e.g., `isAvailabilityRequest()`, `hasConsent`)

## Where to Add New Code

**New Feature (endpoint + logic):**
- Route handler: `src/routes/{domain}.js` (e.g., `src/routes/vendorFeeds.js`)
- Business logic: `src/core/{featureName}.js` (e.g., `src/core/celebrationScheduler.js`)
- Database: Add migration in `migrations/{NNN}_{description}.js`

**New Component/Module:**
- Implementation: `src/core/{moduleName}.js` (exports functions/classes)
- Tests: `tests/{moduleName}.test.js`

**New Integration (external API):**
- Client module: `src/core/{serviceName}.js` (e.g., `src/core/zenoti.js`)
- Helpers: `src/core/{serviceName}*.js` for related utilities (e.g., `zenotiSync.js`, `zenotiAvailability.js`)
- Middleware/route: Add route to `src/routes/integrations.js` or create new file if significant

**Utilities:**
- Shared helpers: `src/utils/{utility}.js` (if used by multiple modules)
- Or inline in `src/core/{moduleName}.js` if single-use

**New Page (manager dashboard):**
- Route handler: `src/routes/{domain}.js`
- Template: Server-rendered HTML in route (use `pageShell.js` for consistent layout)
- Client JS: Add to `public/admin.js` or create `public/{feature}.js`
- Tests: `tests/routes/{domain}.test.js`

## Special Directories

**`migrations/`:**
- Purpose: Versioned database schema changes
- Generated: Yes (created during development)
- Committed: Yes (always)
- Pattern: Run sequentially; track applied state in `schema_migrations` table; idempotent (use `IF NOT EXISTS` / `IF NOT NULL`)

**`public/`:**
- Purpose: Static files served as-is
- Generated: No (manually created/uploaded)
- Committed: Selectively (logos yes, user uploads no)
- Note: `uploads/` should be a symlink or separate mount point for large files

**`data/` (production only):**
- Purpose: SQLite database storage (mounted volume)
- Generated: Yes (by DB migrations on first run)
- Committed: No (PII + operational data)
- Location: `/data/postly.db` (production) or `/tmp/postly.db` (staging)

**`tests/`:**
- Purpose: Unit and integration tests
- Generated: No (committed with code)
- Committed: Yes
- Runner: Jest (configured in `package.json`)

**`docs/` and `schema/` (reference only):**
- Purpose: Documentation and schema reference
- Generated: Manual updates
- Committed: Yes
- Note: CLAUDE.md is the system master reference; update there first

---

*Structure analysis: 2026-03-19*
