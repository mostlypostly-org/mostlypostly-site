# Architecture

**Analysis Date:** 2026-03-19

## Pattern Overview

**Overall:** Layered, multi-tenant SaaS with synchronous request-response and background scheduling

**Key Characteristics:**
- Monolithic Express.js server with SQLite persistent storage
- Request-driven message pipeline (stylist SMS/Telegram → AI → approval → publish)
- Background scheduler polling database for scheduled posts
- Multi-tenant via salon-aware queries (salon_id in session)
- Synchronous DB operations (`better-sqlite3`, no async/await on DB calls)
- External integrations via publishers (Facebook, Instagram, GMB, Zenoti)

## Layers

**Request Handler Layer:**
- Purpose: Accept inbound messages, route to processors, serve manager portal
- Location: `src/routes/`
- Contains: Route handlers for auth, dashboard, integrations, webhooks (Twilio, Telegram, Stripe)
- Depends on: Core logic, storage, publishers
- Used by: Express app, external webhooks (Twilio, Stripe)

**Message Processing Layer:**
- Purpose: Parse stylist messages, classify intent, trigger AI, handle responses
- Location: `src/core/messageRouter.js`
- Contains: Message routing logic, command parsing (APPROVE, EDIT, REDO, JOIN, availability requests)
- Depends on: AI generation, storage, salon lookup, availability sync
- Used by: Twilio/Telegram route handlers via `handleIncomingMessage()`

**Business Logic Layer (Core):**
- Purpose: Implement domain features (caption generation, image building, availability detection, vendor campaigns, tracking)
- Location: `src/core/`
- Contains: ~40 modules implementing individual features (composeFinalCaption, buildAvailabilityImage, celebrationScheduler, venorScheduler, zenotiSync, etc.)
- Depends on: Storage, external APIs (OpenAI, Pexels, Zenoti), publishers
- Used by: Message router, scheduler, route handlers

**Storage Layer:**
- Purpose: Persist state and execute queries
- Location: `db.js`, `src/core/storage.js`
- Contains: Database connection singleton, schema migrations, query helpers
- Depends on: `better-sqlite3`
- Used by: Every layer (synchronous DB.prepare() calls)

**Publisher Layer:**
- Purpose: Send posts to external platforms
- Location: `src/publishers/`
- Contains: Facebook Graph API, Instagram Graph API, Google Business Profile API implementations
- Depends on: Network (fetch), salon config from DB
- Used by: Scheduler, one-off publish handlers

**UI Layer:**
- Purpose: Serve HTML pages and static assets
- Location: `src/ui/` (pageShell component), `public/` (static), `src/routes/` (page routes)
- Contains: Server-rendered HTML with Tailwind CDN, client-side JS (modal system, drag-and-drop)
- Depends on: Storage (for salon names, post data), session auth
- Used by: Managers, stylists, admins

**Scheduler Layer:**
- Purpose: Periodically poll posts table and publish when ready
- Location: `src/scheduler.js`, `src/core/vendorScheduler.js`, `src/core/celebrationScheduler.js`
- Contains: Post enqueueing, publication, retry logic, content-type priority
- Depends on: Storage, publishers, error translator
- Used by: Background tick timer (every N seconds)

## Data Flow

**Stylist Messaging Flow:**

1. Stylist texts photo (SMS/Telegram)
2. Webhook handler (`/inbound/twilio` or `/inbound/telegram`) receives request
3. `messageRouter.handleIncomingMessage()` routes by intent:
   - Text + photo → `generateCaption()` (OpenAI Vision)
   - Approval keywords (APPROVE, EDIT, REDO) → update draft, manage approval state
   - Availability request → `zenotiSync.generateAndSaveAvailabilityPost()`
   - JOIN keyword → multi-turn onboarding conversation
4. Draft stored in in-memory `drafts` Map (keyed by chat ID)
5. Stylist previews caption → approves or redoes
6. If manager approval required: post saved as `manager_pending`, link sent to manager
7. Manager approves via web dashboard → `handleManagerApproval()` calls `enqueuePost()`
8. Post added to `manager_approved` status with `scheduled_for` timestamp

**Publishing Flow:**

1. Scheduler ticks every N seconds (`runSchedulerOnce()`)
2. Queries: all `manager_approved` posts where `scheduled_for <= now()`
3. Checks: posting window, daily caps, stylist fairness
4. Publishes to Facebook → stores `fb_post_id`
5. Publishes to Instagram → stores `ig_media_id`
6. Publishes to GMB (if enabled) → stores `google_post_id`
7. Updates post status to `published`, sets `published_at`
8. Scheduled job `syncSalonInsights()` later pulls reach/likes/reactions/saves from platforms

**Vendor Campaign Flow:**

1. Vendor submits CSV → Platform Console (`/internal/vendors`)
2. CSV parsed → campaigns inserted into `vendor_campaigns`
3. Salon approves brand in Admin → feed enabled in `salon_vendor_feeds`
4. Daily vendor scheduler checks: any new campaigns eligible for this salon?
5. For each eligible campaign: AI generates adapted caption, builds post, enqueues
6. Captions use branded hashtag block (not passed to AI, appended after)
7. Affiliate URL injected in prompt if set per salon+vendor
8. Posts published per normal scheduler rules

**Availability Request Flow:**

1. Stylist texts "Post my availability" (no photo)
2. `messageRouter` detects intent via `isAvailabilityRequest()`
3. Check Zenoti integration:
   - If connected + stylist mapped: `zenotiSync.getPooledStylistSlots()` fetches cached working hours (30-min pool)
   - If pool stale: `syncAvailabilityPool()` hits Zenoti API, caches all mapped stylists' 14-day blocks
   - Parse date hint if present (today/tomorrow, this/next week, specific date)
4. Generate availability image via `buildAvailabilityImage()` with stylist photo + slots
5. Save post as `manager_pending`
6. Send stylist a link to review and approve

**Analytics Sync Flow:**

1. Dashboard calls `GET /api/analytics/sync` (manual trigger)
2. Or scheduler tick runs `runCelebrationCheck()` at 6am salon-local time, then `syncSalonInsights()`
3. Queries DB: all published posts without `fetched_at` (or stale insights)
4. Facebook Graph API: `/feed` to list posts, match by timestamp (±5 min), fetch insights
5. Instagram Graph API: `/{igBusinessId}/media` to list media, fetch post insights
6. GMB API: `localPosts` endpoint to fetch impressions + clicks
7. Upsert into `post_insights` with platform field (facebook / instagram / google)

**State Management:**

- **Session state** (browser auth): `req.session.manager_id`, `req.session.salon_id`, `req.session.group_id`
- **Draft state** (in-memory): `drafts` Map (per stylist chat ID) holds caption + metadata until approval
- **In-memory caches**: Zenoti availability pool (30-min TTL), font files (Google Font TTF base64), leaderboard calculations
- **DB state** (persistent): posts, stylists, managers, salons, integrations, vendor campaigns, tracking tokens, insights

## Key Abstractions

**Multi-Tenant Lookup:**
- Purpose: Identify salon and stylist from inbound message
- Examples: `src/core/salonLookup.js`
- Pattern: First look up stylist by phone/chat_id, then salon_id via FK

**Caption Composition:**
- Purpose: Build final captions with platform-specific variants (FB vs IG)
- Examples: `src/core/composeFinalCaption.js`, `src/core/toneVariants.js`
- Pattern: AI generates base caption, then split at publish time — FB gets full caption with URL, IG strips URL and uses @handle

**Image Building:**
- Purpose: Generate 1080×1920 story images with real photos + SVG overlays
- Examples: `src/core/buildAvailabilityImage.js`, `src/core/buildPromotionImage.js`, `src/core/celebrationImageGen.js`
- Pattern: Photo-first fallback chain (stylist photo → stock photos → Pexels → solid color); frosted glass overlay with text/slots

**Zenoti Integration:**
- Purpose: Fetch stylist working hours and appointments for availability posts
- Examples: `src/core/zenoti.js`, `src/core/zenotiSync.js`, `src/core/zenotiAvailability.js`
- Pattern: Sync pooled slots (30-min cache), categorize by appointment history, format readable blocks

**Tracking Tokens:**
- Purpose: Generate short URLs for booking/vendor link tracking
- Examples: `src/core/trackingUrl.js`, `src/core/utm.js`
- Pattern: `buildTrackingToken()` inserts into `utm_clicks` table, returns 8-char opaque token; `/t/:token` redirects with logged click

**Gamification:**
- Purpose: Track stylist performance and reward consistency
- Examples: `src/core/gamification.js`, `src/routes/leaderboard.js`
- Pattern: Weekly streaks, point scoring, leaderboard rankings per salon

## Entry Points

**Web Server (`server.js`):**
- Location: `server.js`
- Triggers: Node.js startup
- Responsibilities: Express app initialization, middleware setup, session store, CSRF protection, route registration, scheduler startup

**Message Inbound (`/inbound/twilio`, `/inbound/telegram`):**
- Location: `src/routes/twilio.js`, `src/routes/telegram.js`
- Triggers: Twilio/Telegram webhook (stylist texts a message)
- Responsibilities: Verify signature, extract message + media, call `messageRouter.handleIncomingMessage()`

**Webhook (`/billing/webhook`):**
- Location: `src/routes/billing.js`
- Triggers: Stripe webhook (subscription events)
- Responsibilities: Verify signature, update salon billing status, upsert subscription data

**Manager Auth (`/manager/login`, `/manager/signup`):**
- Location: `src/routes/managerAuth.js`
- Triggers: Manager visits login/signup page
- Responsibilities: Token email delivery, password reset, session establishment

**Scheduler Tick:**
- Location: `src/scheduler.js`
- Triggers: Background timer (every 10–30 seconds)
- Responsibilities: Query posts table, check windows/caps, publish, update status, retry failures

**Vendor Scheduler Tick:**
- Location: `src/core/vendorScheduler.js`
- Triggers: Startup + daily timer
- Responsibilities: Find eligible vendor campaigns, generate posts, enqueue for publishing

**Celebration Scheduler Tick:**
- Location: `src/core/celebrationScheduler.js`
- Triggers: Scheduler tick at 6am salon-local time each day
- Responsibilities: Find birthdays/anniversaries, generate dual-format images, notify manager

**Google OAuth Callback (`/auth/google/callback`):**
- Location: `src/routes/googleAuth.js`
- Triggers: User returns from Google's OAuth approval flow
- Responsibilities: Exchange code for GMB tokens, store refresh token, enable GMB publishing

## Error Handling

**Strategy:** Try-catch at route/message boundaries; errors translated to plain English; failed posts retry with exponential backoff

**Patterns:**

- **API Errors (Facebook/Instagram/Zenoti):** Wrapped in try-catch, logged with context, user-facing message via `postErrorTranslator.translatePostError()`
- **Token Expiry (GMB):** `googleTokenRefresh()` silently refreshes before publish if within 5 min of expiry
- **Image Generation Failures:** Silent fallback to solid color (Pexels timeout → solid background)
- **Failed Posts:** Status set to `failed`, max 3 retries, SMS to managers with error text
- **Validation Errors:** Form submissions validated server-side; CSRF token checked on all POST endpoints
- **Database Errors:** Caught in middleware, logged, generic response sent (no DB details leaked)

## Cross-Cutting Concerns

**Logging:** Console.log with prefixes (`[Router]`, `🚀`, `❌`) for manual log analysis. `analyticsDb.js` tracks audit events (post creation, approval, publish, sync). No centralized logger configured.

**Validation:**
- Salon queries always filter by `req.session.salon_id` (IDOR guard)
- Stylist phone/chat_id validated on inbound messages
- All external URLs validated (media proxy checks `https://api.twilio.com` prefix)
- Email format checked via regex on auth forms

**Authentication:**
- Managers: Email+password (bcrypt hashed) OR token-based login links (7-day expiry, SMS delivery)
- Stylists: No auth — SMS/Telegram inbound only, token-based portal access (24-hour expiry per post)
- Admin (Platform Console): `INTERNAL_SECRET` env var + `INTERNAL_PIN` session two-factor

**Authorization:**
- Manager role system: `owner` (billing+admin), `manager` (team/branding), `staff` (dashboard/analytics only)
- Portal access: Grant via Team page (associates stylist to manager account)
- Vendor approvals: Platform Console reviews requests; salon can't see campaign until approved

---

*Architecture analysis: 2026-03-19*
