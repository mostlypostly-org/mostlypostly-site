# External Integrations

**Analysis Date:** 2026-03-19

## APIs & External Services

**Messaging (Primary):**
- Twilio SMS/MMS — Stylist inbound message channel; manager/stylist SMS notifications
  - SDK: `twilio` npm package v5.10.4
  - Auth: `TWILIO_ACCOUNT_SID` + `TWILIO_AUTH_TOKEN` + `TWILIO_PHONE_NUMBER` (or `TWILIO_MESSAGING_SERVICE_SID`)
  - Webhook: `POST /twilio/inbound` — validates Twilio request signature
  - Route: `src/routes/twilio.js`
  - Features: `sendViaTwilio()`, `sendViaRcs()` with RCS chip support

**Messaging (Secondary):**
- Telegram Bot API — Alternative stylist channel
  - Auth: `TELEGRAM_BOT_TOKEN` in headers
  - Webhook: `POST /telegram/webhook`
  - Route: `src/routes/telegram.js`
  - Uses raw fetch to `https://api.telegram.org/bot{token}/sendMessage`

**AI & Content Generation:**
- OpenAI GPT-4o-mini — Caption generation, post type classification
  - SDK: `openai` npm package v6.8.1
  - Auth: `OPENAI_API_KEY` (Bearer token)
  - Route: `src/openai.js`
  - Vision support: Analyzes MMS/uploaded images for context
  - Used in: `generateCaption()` function accepts base64 image data

**Image Search (Fallback):**
- Pexels API — Real photo backgrounds when no stock photos available
  - Auth: `PEXELS_API_KEY` in Authorization header
  - Endpoint: `https://api.pexels.com/v1/search`
  - Route: `src/core/pexels.js`
  - Returns: Portrait-oriented large images (1080x1920 compatible)

## Data Storage

**Databases:**
- SQLite 3 via better-sqlite3
  - File: `/data/postly.db` (production) or `./postly.db` (local)
  - No remote database; single-file persistent storage
  - Backup: Manual (file-based, no cloud sync configured)

**File Storage:**
- Local filesystem only
  - Uploaded logos: `public/uploads/` (manager signup, admin panel)
  - Generated images: Temporary files in OS tmp directory, then served via Express
  - Twilio MMS URLs: Expire ~90 days; proxied through `/api/media-proxy` for browser rendering

**Caching:**
- In-memory: Salon data (local `APP_ENV=local` uses `salons/` JSON files + chokidar watcher)
- No Redis or distributed cache

## Authentication & Identity

**Auth Provider:**
- Custom implementation (no OAuth for manager login)
  - `src/routes/managerAuth.js` — Email/password signup, login token links, password reset
  - Session-based: Express sessions stored in SQLite
  - Login tokens: 7-day expiry, sent via SMS
  - Password reset tokens: 45-minute expiry, sent via SMS

**Third-Party OAuth:**
- Facebook Graph API v19.0+
  - Endpoint: `https://graph.facebook.com/v19.0/`
  - OAuth flow: `src/routes/facebookAuth.js`
  - Token type: Long-lived page token (stored in `salons.facebook_page_token`)
  - Never expires; refreshed silently before publishing

- Instagram Graph API v19.0+
  - OAuth flow: Part of Facebook OAuth (Instagram accounts linked to FB business accounts)
  - Token: Inherited from Facebook page token
  - Endpoint: `https://graph.instagram.com/v19.0/`

- Google Business Profile (GMB)
  - OAuth 2.0 flow: `src/routes/googleAuth.js`
  - Auth: `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` + `GOOGLE_REDIRECT_URI`
  - Tokens: `google_access_token` (short-lived, ~1 hour) + `google_refresh_token` (long-lived)
  - Refresh logic: `src/core/googleTokenRefresh.js` — auto-refresh if expired or within 5 min of expiry
  - Endpoint: `https://mybusiness.googleapis.com/v4/`
  - Route: `src/routes/googleAuth.js`, `src/publishers/googleBusiness.js`

## Monitoring & Observability

**Error Tracking:**
- None configured; errors logged to stdout/stderr

**Logs:**
- Console output (stdout) — captured by Render logs
- `src/utils/logHelper.js` — createLogger utility for prefixing
- App, scheduler, moderation, and analytics logs can have different prefixes
- No centralized log aggregation (Datadog, Sentry, etc.)

**Debugging:**
- `POST /analytics/debug?salon=<slug>` — Tests FB token, FB insights, IG account, IG media list, IG post insights
- `POST /analytics/reset-and-backfill-fb-ids?salon=<slug>` — Clears and re-syncs FB post IDs
- `POST /analytics/sync?salon=<slug>` — Manual insight sync trigger
- Routes: `src/routes/analytics.js`

## Publishing & Distribution

**Facebook:**
- Endpoint: `https://graph.facebook.com/v19.0/{pageId}/photos` (photo upload)
- Fallback: `https://graph.facebook.com/v19.0/{pageId}/feed` (text-only)
- Multi-photo: `attached_media` array
- Route: `src/publishers/facebook.js` exports `publishToFacebook()`, `publishToFacebookMulti()`

**Instagram:**
- Endpoint: `https://graph.instagram.com/v19.0/{igBusinessId}/media` (create media container)
- Publish: Post media container to feed
- Route: `src/publishers/instagram.js` exports `publishToInstagram()`
- Limitation: Direct `/{mediaId}?fields=` fails if token is from different OAuth connection; uses media list + timestamp matching (±5 min)

**Google Business Profile:**
- Endpoint: `https://mybusiness.googleapis.com/v4/{locationId}/localPosts`
- Post types: `topicType = "STANDARD"` (what's new) or `"OFFER"` (promotion)
- Route: `src/publishers/googleBusiness.js` exports `publishWhatsNewToGmb()`, `publishOfferToGmb()`

## Billing & Payments

**Stripe:**
- SDK: `stripe` npm package v20.4.1 (API version 2024-06-20)
- Auth: `STRIPE_SECRET_KEY` (sk_live_... in production, sk_test_... in staging)
- Webhook: `POST /billing/webhook` — events: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
- Webhook secret: `STRIPE_WEBHOOK_SECRET` (whsec_...)
- Checkout: Creates Stripe session with plan pricing IDs
- Portal: Linked from billing page for payment method + invoice management
- Promo codes: `allow_promotion_codes: true` in all Stripe sessions; codes managed in Stripe Dashboard
- Route: `src/routes/billing.js`

**Plan Pricing:**
| Plan | Monthly | Annual |
|---|---|---|
| Starter | $49 | $528 |
| Growth | $149 | $1,608 |
| Pro | $249 | $2,688 |

## Email

**Transactional Emails:**
- Resend (re_... API keys)
- Auth: `RESEND_API_KEY` env var
- From: `EMAIL_FROM` (default: hello@mostlypostly.com; must be verified domain in Resend)
- Route: `src/core/email.js` — `sendEmail()`, `sendVerificationEmail()`, `sendWelcomeEmail()`, `sendCancellationEmail()`
- Uses: Signup verification, welcome, password reset, subscription cancellation

## Salon Software Integration

**Zenoti (POS):**
- Endpoint: `https://api.zenoti.com/v1/`
- Auth: `apikey {api_key}` header + `application_id` header
- DB table: `salon_integrations` — stores `api_key`, `app_id`, `center_id`, `sync_enabled`, `last_event_at`
- Endpoints used:
  - `GET /Centers/{centerId}/employee_schedules` — Working hours
  - `GET /appointments` — Max 10-day range; auto-chunks longer ranges
  - `GET /Centers/{centerId}/services` — Service catalog (keyword-inferred categories)
  - `GET /centers/{centerId}/employees` — Stylist list
- Route: `src/core/zenoti.js` — client creation and API calls
- Availability: `src/core/zenotiSync.js` — in-memory 30-min pool to batch requests
- SMS-triggered availability posts use 14-day window or parsed date range

## Webhooks & Callbacks

**Incoming Webhooks (Public, no auth):**
- `POST /twilio/inbound` — Twilio SMS/MMS webhook (signature validation required)
- `POST /telegram/webhook` — Telegram message webhook
- `POST /billing/webhook` — Stripe webhook (signature validation required)

**Outgoing Webhooks:**
- None configured; no push notifications to external services
- Post publishing sends to Facebook/Instagram/GMB APIs directly (not webhooks)

## Tracking & Analytics

**Tracking Links:**
- UTM parameter injection: `src/core/utm.js`
- Booking links: Replaced with short tokens `/t/{token}` in captions during publish
- Vendor affiliate URLs: Injected into AI prompt; tracked separately
- Bio links: Permanent `/t/{slug}/book` (logs every click)
- DB table: `utm_clicks` — logs token, click_type, destination, clicked_at, IP hash (SHA-256, never raw IP)
- Route: `src/routes/tracking.js` — `GET /t/:token` (302 redirect, first click), `GET /t/:slug/book` (307, every click)

**Insights Sync:**
- Facebook Insights: `POST /analytics/sync` calls `syncSalonInsights()`
- Instagram Insights: Via Facebook Graph API (IG metrics returned in FB insights query)
- GMB Insights: `src/core/fetchGmbInsights.js` — calls GMB `localPosts:reportInsights`
- DB table: `post_insights` — reach, impressions, likes, comments, shares, saves, engagement_rate
- Upsert on conflict: UNIQUE(post_id, platform)

---

*Integration audit: 2026-03-19*
