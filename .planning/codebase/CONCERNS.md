# Codebase Concerns

**Analysis Date:** 2026-03-19

## Tech Debt

### Dual Encryption Implementation

**Issue:** Two separate encryption modules exist with inconsistent naming: `src/core/encrypt.js` and `src/core/encryption.js`. Both implement AES-256-GCM with different env var names (`ENCRYPTION_KEY` vs `SALON_POS_ENCRYPTION_KEY`).

**Files:**
- `src/core/encrypt.js` (uses `ENCRYPTION_KEY`)
- `src/core/encryption.js` (uses `SALON_POS_ENCRYPTION_KEY`)

**Impact:** Maintainability nightmare. Future developers won't know which to use. If one is updated for security, the other might not be. Integration credentials may be encrypted with wrong module.

**Fix approach:** Audit which module is actively used in codebase, consolidate into single canonical module `src/core/encryption.js`, remove unused one. Standardize on `ENCRYPTION_KEY` env var. Verify all credential storage uses the same module.

---

### In-Memory State Without Persistence

**Issue:** `src/core/joinSessionStore.js` stores signup conversation state entirely in a Map. Server restart or Render auto-deploy wipes all active join conversations mid-flow.

**Files:** `src/core/joinSessionStore.js`, `src/core/messageRouter.js:1200+`

**Impact:** Stylists mid-join (phone verification, confirmation) lose context on server restart. Must restart entire signup sequence. No graceful recovery.

**Fix approach:** Migrate session state to persistent DB table `join_sessions` with `chat_id, step, data, created_at, ttl`. Implement auto-cleanup for expired sessions. Test multi-server deployments don't collide.

---

### Debug Logging Left in Production Code

**Issue:** Extensive `console.log()` statements throughout codebase with debug labels ("🧨 FORCE DEBUG", "🔐", "🕓", etc.) that increase noise and expose internal state.

**Files:**
- `src/core/messageRouter.js` (lines 956–959, 1028–1029, 1057, 1081, 1113, 1116, 1157, etc.) — 50+ debug logs
- `src/core/zenoti.js` (lines 79, 86)
- `src/scheduler.js`
- `src/openai.js`

**Impact:** Log volume makes real errors harder to spot in production. Potential for sensitive data leakage if debug output contains PII (phone numbers, emails).

**Fix approach:** Implement structured logging with log levels (DEBUG, INFO, WARN, ERROR). Gate debug statements behind `LOG_LEVEL` environment variable. Use `process.env.APP_ENV !== "production"` for development-only logs.

---

### Incomplete TODOs in Live Code

**Issue:** Multiple TODO/FIXME comments indicate unfinished work paths in production code.

**Files:**
- `src/routes/managerAuth.js:1039` — "TODO: replace with email or SMS delivery" (password reset link logged to console)
- `src/core/integrationHandlers.js:6` — "TODO: handle appointment.booked, appointment.cancelled, etc." (Zenoti webhooks logged but not processed)
- `src/core/integrationHandlers.js:11` — "TODO: handle vagaro webhook events" (Vagaro stubs)

**Impact:** Password reset tokens leak to console logs. Integration events silently ignored. Managers have no way to use features marked TODO.

**Fix approach:** Complete password reset delivery (Resend already imported). Implement Zenoti webhook handlers. Document why Vagaro is stubbed (future feature).

---

### Single-Process Puppeteer with Memory Constraints

**Issue:** `src/core/puppeteerRenderer.js:39–50` launches Puppeteer with `--single-process --no-zygote` flags optimized for Render Starter's 512MB RAM. This disables process isolation.

**Files:** `src/core/puppeteerRenderer.js`

**Impact:** Process isolation disabled. If rendering fails or page is malicious (SVG injection), entire Node process crashes. Memory leaks in browser instance can OOM server.

**Fix approach:** Monitor rendering memory usage. Implement page timeout + process restart cycle after N renders. Consider upgrade to Render Standard tier (1GB) or separate rendering service.

---

### BROKEN_IMG_PLACEHOLDER Duplication

**Issue:** Image error handling logic duplicated in two places with brittle inline event handlers (`onload`/`onerror` strings embedded in HTML).

**Files:**
- `src/routes/stylistPortal.js:105–125` (BROKEN_PORTAL)
- `src/routes/manager.js:44–85` (BROKEN_IMG_PLACEHOLDER)

**Impact:** Image expiry fallback duplicated. Client-side error handling tied to HTML generation. Hard to test and maintain.

**Fix approach:** Extract to shared utility module. Implement image preload check server-side. Use progressive enhancement (fetch JSON to verify URL validity before rendering).

---

### Password Reset Token Exposure in Console

**Issue:** `src/routes/managerAuth.js:1039–1042` logs full password reset URL to console without ENV guard. Code exists in production.

**Files:** `src/routes/managerAuth.js:1039`

**Impact:** If Render logs are accessed, reset tokens are visible with no expiry guard in logging.

**Fix approach:** Add `APP_ENV !== "production"` guard. Implement Resend email delivery (already imported) instead of console.log. Never log sensitive tokens.

---

## Known Bugs

### Manager Approval Flag Resolution Ambiguity

**Symptoms:** Multiple salon record lookups return inconsistent `require_manager_approval` values. Debug logs in `src/core/messageRouter.js:956–959` show fields from different sources.

**Files:** `src/core/messageRouter.js:938–960`, `src/core/salonLookup.js`

**Trigger:** Manager creates post via SMS with conflicting salon flags from incomplete data migration.

**Workaround:** Check `/analytics/debug?salon=<slug>` endpoint to inspect resolved salon object.

**Root cause:** Salon data can come from JSON file (local dev) or DB (staging/prod). Migration 001 baseline may not have backfilled consistently.

---

### Zenoti Event Handlers Are Stubs

**Symptoms:** Zenoti webhook events logged but never processed. Stylists' availability doesn't auto-update when appointments booked/cancelled in Zenoti.

**Files:** `src/core/integrationHandlers.js:4–7`

**Trigger:** Stylist books appointment in Zenoti; MostlyPostly availability posts show old slots.

**Workaround:** Manual sync via `/manager/integrations/sync-zenoti` (user-initiated, not automatic).

**Impact:** Pro plan feature (Zenoti integration) doesn't deliver real-time updates.

---

### Stylist Portal Image Expiry With No Recovery

**Symptoms:** Twilio MMS images expire after ~90 days. Stylist portal shows expired placeholder, no way to re-request.

**Files:** `src/routes/stylistPortal.js:105–126`

**Trigger:** Stylist opens draft from >90 days ago; image shows as expired.

**Workaround:** Stylist must re-upload photo.

**Impact:** Old drafts become unusable. No audit trail of what original image was.

---

### UTM Tracking URL Injection Failure Doesn't Block Post

**Issue:** In `src/core/composeFinalCaption.js`, if `buildTrackingToken()` throws error, function catches silently and falls back to raw booking URL. Only logged.

**Files:** `src/core/composeFinalCaption.js` (lines ~150–160)

**Impact:** Post publishes with untracked booking URL. Analytics ROI calculation becomes inaccurate. Manager sees no indication that tracking failed.

**Fix approach:** Propagate tracking errors upstream. If token generation fails, pause post and SMS manager the reason. Don't silently degrade.

---

### Image Generation Timeouts in Puppeteer

**Issue:** `src/core/puppeteerRenderer.js` uses `Promise.race()` with 6-second timeout for Google Fonts. If Chrome fails to launch, fallback isn't clean.

**Files:** `src/core/puppeteerRenderer.js` (lines 13–31, 73–78)

**Impact:** Image generation fails with cryptic error. Post sits in `manager_approved` state. Scheduler retry logic kicks in but no user visibility.

**Fix approach:** Add structured error handling in scheduler. If 2 retries fail, downgrade image quality (skip fonts, use browser defaults). SMS manager with clear error.

---

### Vendor Post Logging Without Timestamp Enforcement

**Symptoms:** `vendor_post_log` tracks posts per campaign per month, but month boundaries are ISO format. Month-end UTC boundary may cause double-count or skip.

**Files:** `src/core/vendorScheduler.js` (post log insert)

**Trigger:** Scheduler runs 11:55 PM UTC on last day of month; post scheduled for next day.

**Workaround:** Frequency cap is soft limit (can overshoot by 1 per month).

**Risk level:** Low (affects vendor compliance reports, not critical path).

---

## Security Considerations

### IDOR Vulnerability on Route Handlers

**Issue:** While multi-tenancy awareness documented, audit shows inconsistent application. Some routes check `req.session.salon_id`, others don't filter queries properly.

**Files:** All route files, especially newer ones like `src/routes/vendorFeeds.js`, `src/routes/vendorAdmin.js`

**Impact:** Manager from salon A could view/edit salon B's posts, vendor feeds, or integrations by guessing resource IDs.

**Fix approach:** Create middleware `requireSalonOwnership(req, resource)` to validate `req.session.salon_id` against `resource.salon_id`. Audit every route with ID params. Add integration tests for IDOR.

---

### Webhook Signature Verification Inconsistency

**Issue:** Twilio webhook validates signatures only if `TWILIO_AUTH_TOKEN` is set. Zenoti webhooks check signature but schema doesn't mandate `webhook_secret` at DB level.

**Files:** `src/routes/twilio.js` (line 70), `src/routes/integrations.js` (line 905)

**Impact:** In development without tokens, webhooks unverified. Attacker could spoof as real Twilio/Zenoti. In production, NULL `webhook_secret` skips Zenoti validation entirely.

**Fix approach:** Make webhook secrets mandatory at integration setup. Validate signature unconditionally. Fail request if token env var missing. Add admin console to rotate secrets.

---

### Image Proxy Lacks Rate Limiting

**Risk:** `/api/media-proxy` endpoint not rate-limited. Attacker can hammer endpoint to exhaust bandwidth or DOS Twilio API.

**Files:** `src/routes/analytics.js:53–70` (toProxyUrl function referenced throughout)

**Current mitigation:** Proxy only handles Twilio URLs (validated with regex), but no per-user or per-IP rate limit.

**Recommendations:**
1. Add dedicated rate limiter for `/api/media-proxy` (e.g., 100 requests/min per IP)
2. Implement URL signature validation
3. Cache media locally with TTL instead of proxying on every request

---

### Puppeteer SVG Injection Risk

**Risk:** `src/core/buildPromotionImage.js`, `src/core/buildAvailabilityImage.js`, `src/core/celebrationImageGen.js` generate SVG overlays with brand colors and salon names. Unescaped salon name could inject SVG/CSS.

**Files:** `src/core/buildPromotionImage.js`, `src/core/buildAvailabilityImage.js` (SVG template construction)

**Current mitigation:** None visible. SVG strings are template literals with direct variable insertion.

**Recommendations:**
1. Sanitize all user input before SVG injection (salon name, brand colors, hashtags)
2. Use XML escaping for text nodes
3. Validate brand_palette hex colors with regex before insertion

---

### Zenoti API Credentials Storage

**Risk:** `salon_integrations.api_key` stores Zenoti API key. Claims to be encrypted via `encryption.js` but should verify application.

**Files:** `src/routes/integrations.js:630` (encryption call), `src/core/zenoti.js:32–36` (header auth)

**Current mitigation:** Encryption.js AES-256-GCM wrapper claims encryption at rest.

**Recommendations:**
1. Verify encryption applied on every INSERT/UPDATE to `salon_integrations`
2. Ensure `ENCRYPTION_KEY` rotation on security incident
3. Add audit log entry for every API call using external credentials

---

### Google OAuth Tokens at Risk of Expiry Race

**Risk:** `src/core/googleTokenRefresh.js` silently refreshes GMB token if within 5 minutes of expiry. If refresh fails, next publish uses expired token and fails silently.

**Files:** `src/core/googleTokenRefresh.js`, `src/scheduler.js:515–520`

**Current mitigation:** Publish failure triggers retry logic. Debug endpoint exists.

**Recommendations:**
1. Log refresh failures with salon context for support team
2. Send SMS alert to manager if GMB publish fails due to token expiry
3. Extend refresh window to 30 minutes to reduce race condition

---

### Rate Limiting on Public Tracking Link

**Risk:** `src/routes/tracking.js` `/t/:token` endpoint has no rate limiting. Attacker can spam clicks to inflate metrics or cause DB bloat.

**Files:** `src/routes/tracking.js`

**Impact:** Analytics become unreliable. DB grows unbounded with garbage clicks. No coordinated click fraud detection.

**Fix approach:** Apply express-rate-limit (100 clicks/minute per IP). Implement click deduplication by IP + user-agent within 5-second window. Add anomaly detection in analytics.

---

### Manager MFA Backup Codes Stored as JSON

**Risk:** `manager_mfa.backup_codes` column stores codes as JSON array. If one code compromised, entire array isn't invalidated individually.

**Files:** `src/routes/mfa.js:223–229`

**Current mitigation:** Codes are bcrypt hashed. Only 8 codes available.

**Recommendations:**
1. Track backup code usage (mark as used after consumption)
2. Warn user when 1–2 codes remain
3. Implement "generate new set" UI to rotate codes

---

## Performance Bottlenecks

### syncSalonInsights() Fetches All Posts Without Pagination

**Problem:** `src/core/fetchInsights.js:215–220` queries `SELECT * FROM posts WHERE salon_id=? LIMIT 100`, then iterates each post calling FB/IG APIs. For 10k+ posts, this creates cascading API calls.

**Files:** `src/core/fetchInsights.js:215–264`

**Cause:** No cursor-based pagination or date-range filtering. Always fetches "most recent 100 posts" (arbitrary).

**Impact:** Salons with >100 posts miss insights on older posts. Sync takes >30s on large catalogs.

**Improvement path:**
1. Add `WHERE published_at > ? AND published_at < ?` to sync only recent window (e.g., last 30 days)
2. Implement pagination cursor to avoid re-fetching
3. Cache last-sync timestamp per salon, skip already-synced posts

---

### Puppeteer Single-Process Instance with Font Loading Race

**Problem:** `src/core/puppeteerRenderer.js:75–77` uses `Promise.race()` with 6-second font load timeout. If CDN slow, renders proceed with fallback; if fast, 6s is wasted.

**Files:** `src/core/puppeteerRenderer.js:75–77`

**Cause:** No adaptive timeout. 6s threshold may be too conservative or aggressive.

**Impact:** Image generation takes minimum 6s per image even if fonts load in 1s. Queued images pile up during high-load periods.

**Improvement path:**
1. Measure font load time per region, adjust timeout by `APP_ENV`
2. Pre-download fonts and inline as base64
3. Use `document.fonts.check()` to verify fonts are loaded

---

### Availability Pool Sync Refreshes All Stylists on Expiry

**Problem:** `src/core/zenotiSync.js` implements 30-minute in-memory pool for stylist availability. When pool expires, entire pool refreshes (all stylists), even if one stylist requested availability.

**Files:** `src/core/zenotiSync.js`, `src/core/messageRouter.js:1323`

**Cause:** Pool keyed by salon, not by stylist. Cache invalidation is all-or-nothing.

**Impact:** During peak hours, multiple stylists texting availability trigger 3+ Zenoti API calls in 5 minutes, each fetching same salon's working hours.

**Improvement path:**
1. Implement per-stylist pool entries (cache by salon + stylist pair)
2. Extend pool TTL to 60 minutes for working hours (they don't change intraday)
3. Add jitter to pool expiry to spread refresh load

---

### Vendor Scheduler Runs Full Campaign Query Daily

**Problem:** `src/core/vendorScheduler.js` iterates all Pro salons with approved vendor feeds, then queries campaigns for each. No caching or deduplication.

**Files:** `src/core/vendorScheduler.js` (daily loop)

**Cause:** Campaigns stored in `vendor_campaigns` table. No derived/summary table for active campaigns.

**Impact:** 50 Pro salons × 5 vendor brands = 250 campaign queries per day. If each query is 100ms, that's 25 seconds total + OpenAI calls.

**Improvement path:**
1. Pre-compute active campaign list once per day, cache to memory
2. Add materialized view of active campaigns grouped by vendor
3. Batch vendor caption generation with parallel OpenAI calls

---

### Scheduler Checks All Salons Every N Seconds

**Problem:** Scheduler loop queries all salons, counts daily posts, checks window. Scales O(n salons).

**Files:** `src/scheduler.js`

**Limit:** At 1000 salons with 10 pending posts each, scheduler loop takes ~2 seconds.

**Improvement path:**
1. Add index on `(salon_id, status, scheduled_for)`
2. Implement "dirty" flag on salons. Skip salons with no pending posts.
3. Split scheduler into per-salon workers once > 500 salons deployed

---

### Zenoti API Calls Per Salon Unparallelized

**Problem:** `syncAvailabilityPool` fetches all stylists' appointments sequentially. Zenoti API calls timeout after 30s.

**Files:** `src/core/zenoti.js`, `src/core/zenotiSync.js`

**Limit:** Salon with 20 stylists = 20 sequential API calls. If each is 1.5s, total is 30s (hits timeout).

**Improvement path:**
1. Parallelize stylist availability fetching (Promise.all instead of sequential)
2. Implement per-stylist caching (only fetch changed appointments)
3. Use Zenoti webhooks (currently TODO) to push appointments instead of polling

---

## Fragile Areas

### Migration 001 Baseline Incomplete

**Files:** `migrations/001_baseline_patches.js`

**Why fragile:** This migration is a catch-all for schema patches. If a new column added and not backfilled, old salons get NULL values causing implicit type coercions.

**Safe modification:** Always backfill new columns in migration scripts. Test migrations on staging DB with real data.

**Test coverage:** No automated migration test suite. Must test manually on staging backup.

---

### messageRouter.js Is 1500+ Lines

**Files:** `src/core/messageRouter.js`

**Why fragile:** Single function handles SMS/Telegram inbound, image processing, AI captioning, draft storage, availability routing, manager approval, Zenoti sync, and publication. Changes to one path break another.

**Safe modification:** Extract flows into separate modules (availabilityFlow, approvalFlow, imagingFlow) with clear input/output contracts.

**Test coverage:** No unit tests visible. Only manual testing via live SMS/Telegram.

**Risk:** Any change to draft storage, approval, or Zenoti routing requires end-to-end SMS testing.

---

### Seller_name vs salon_id Lookup Inconsistency

**Files:** `src/core/salonLookup.js`, various route handlers

**Why fragile:** Some routes accept `salon_id` (slug string), others accept `seller_name` (legacy). Lookups sometimes filter by stylist phone → salon instead of explicit salon_id. If stylist transferred, phone-based lookup breaks.

**Safe modification:** Standardize on explicit `salon_id` parameter in all queries. Update stylist model to require `salon_id + stylist_id` tuple (no phone-based lookup).

**Test coverage:** No tests for multi-salon stylist edge cases.

**Risk:** If `salon_id` can be NULL on stylists table, many queries silently return wrong results.

---

### Brand Palette Extraction (One-Shot)

**Files:** `src/routes/onboarding.js`, `src/core/buildPromotionImage.js`, `src/core/buildAvailabilityImage.js`

**Why fragile:** Brand palette extracted once during onboarding from website HTML via OpenAI. If salon updates website colors, images use old palette. Manual re-extract requires undocumented `/onboarding/brand?reset=1` URL.

**Safe modification:** Add "Re-extract brand colors" button to Admin panel. Schedule periodic re-extract (e.g., monthly) to catch refreshes.

**Test coverage:** No tests for palette extraction or image color accuracy.

**Risk:** Promotion images may look off-brand if salon rebrands.

---

### Retry Logic Uses retry_count Column (May Overflow)

**Files:** `src/scheduler.js:536–542` (retry increment), `src/scheduler.js:256` (query LIMIT)

**Why fragile:** Max retries = 3 hardcoded. When MAX_RETRIES exceeded, post marked `failed` and never retried again. No manual retry UI.

**Safe modification:** Add `attempts` table to track retry attempts with timestamp. Allow managers to manually retry failed posts from Dashboard.

**Test coverage:** No tests for retry exhaustion or manual retry flow.

**Risk:** Failed posts stuck in `failed` state. Manager must use direct DB query to recover.

---

## Scaling Limits

### SQLite Database File-Based on Render Starter

**Current capacity:** Single SQLite file. ~2M posts before disk space becomes constraint (1KB per row metadata).

**Limit:** Render Starter tier has 512MB ephemeral storage. If DB grows to 256MB+, little room for temp files during migrations.

**Scaling path:**
1. Monitor DB file size. Migrate to PostgreSQL hosted on Render when DB > 100MB.
2. Implement archival strategy (move posts older than 2 years to cold storage).
3. Add read replicas for analytics once scaling warrants it.

---

### Twilio Image Proxy Has No Caching

**Current capacity:** Twilio MMS URLs expire after ~90 days. Proxy endpoint not rate-limited or cached.

**Limit:** Each image view (dashboard, analytics, portal) makes request to `/api/media-proxy`. 100 views = 100 Twilio API calls.

**Scaling path:**
1. Implement in-memory LRU cache for proxied images (cache 24h)
2. Add Redis cache layer once 100+ concurrent users
3. Implement local image rehosting (download Twilio URL on post creation)

---

### Puppeteer Single-Browser with --single-process

**Current capacity:** One browser instance handles all concurrent rendering. Queued if 2+ requests arrive simultaneously.

**Limit:** High-concurrency periods (promotions auto-generate, availability bulk-posts) create image generation queue backlog. Single-process increases memory pressure linearly.

**Scaling path:**
1. Implement image generation worker queue (Bull/BullMQ with Redis)
2. Move Puppeteer to dedicated rendering service (Render "Private Space")
3. Monitor rendering latency. Alert if queue depth > 10 items.

---

## Dependencies at Risk

### Puppeteer as Critical Dependency

**Risk:** Puppeteer ^24.39.1. Chrome binary installed as postinstall step but not verified on startup. If Chrome fails to install, entire image generation breaks.

**Files:** `package.json`, `src/core/puppeteerRenderer.js`, postinstall script

**Impact:** Server startup fails silently. Image generation fails at runtime. Render deployment could fail without clear error.

**Fix approach:** Add startup health check in `server.js` that attempts Puppeteer launch. Fail fast with clear error if Chrome unavailable. Implement fallback to headless SVG-only images.

---

### bcryptjs Hanging on Large Passwords

**Risk:** Dependency `bcryptjs` ^3.0.3 has slow performance with passwords >70 characters. Signup form doesn't enforce password length limit.

**Files:** `src/routes/managerAuth.js`, `package.json`

**Impact:** Attacker sends 1000+ character password, causing bcrypt to hang minutes. DoS vector.

**Fix approach:** Add password length validation (max 72 characters) in signup form and backend. Consider upgrade to native Node.js crypto (pbkdf2) if bcryptjs becomes deprecated.

---

### sharp ^0.34.5 Memory Leaks

**Risk:** `sharp` library has reported memory leaks in versions <0.35.0. Current pin is ^0.34.5.

**Files:** `package.json`, any image processing code using sharp

**Impact:** Long-running server accumulates memory. Eventually hits Render's 512MB limit and OOM kills process.

**Fix approach:** Upgrade to sharp ^0.35.0+. Monitor memory usage. Implement image processing worker pool with memory limits.

---

### OpenAI API Rate Limits and Pricing

**Risk:** Caption generation calls OpenAI API for every post and vendor campaign. If API down, posting stops. No fallback if API unavailable.

**Files:** `src/core/openai.js`, `src/core/vendorScheduler.js`

**Impact:** At 100 posts/day × $0.003 per caption ≈ $90/month. If API down, zero visibility into fallback.

**Improvement path:**
1. Implement queue + retry with exponential backoff
2. Cache fallback captions in DB (use template if API down)
3. Monitor OpenAI status. Test Claude API as alternative (cheaper).

---

### better-sqlite3 Requires Compilation

**Risk:** `npm install` requires building native bindings. Render's build system may fail if build tools absent.

**Files:** `package.json`

**Impact:** Deployment can fail if Render environment changes. Major version bump requires verification.

**Fix approach:** Lock better-sqlite3 version in package-lock.json. Test version bumps on staging first. Add Render buildpack verification to CI.

---

### Timezone Library (Luxon) Complexity

**Risk:** Scheduler relies on Luxon DateTime for timezone math. Date range calculations for Zenoti must account for salon timezone.

**Files:** `src/scheduler.js`, `src/core/zenotiSync.js`

**Impact:** Daylight savings time transitions or Luxon bugs cause scheduler to miss posts or double-post.

**Fix approach:**
1. Add timezone-aware tests (spring forward, fall back)
2. Implement manual timezone verification endpoint (`/health/timezones`)
3. Store all `scheduled_for` in UTC, convert display-time only

---

## Missing Critical Features

### No Manual Post Retry UI

**Problem:** Failed posts (status='failed') are stuck. Only way to retry is direct DB query.

**Blocks:** Managers cannot recover from temporary API outages without support intervention.

**Current workaround:** Support team deletes failed post from DB. Stylist must re-create.

**Implementation:** Add "Retry" button on failed posts in Dashboard. Reset status to `manager_approved` and re-queue.

---

### No Image Re-Hosting Strategy

**Problem:** All Twilio MMS URLs expire after ~90 days. Old posts lose images.

**Blocks:** Post audit trail and re-sharing broken. Images unavailable for analytics.

**Current workaround:** None. Old posts show "Image expired".

**Implementation:** On post creation, download Twilio image and store to `public/uploads/` or S3. Point all references to persistent URL.

---

### No Audit Log for Integration Access

**Problem:** Zenoti API key, Google OAuth tokens, Stripe keys accessed but not logged.

**Blocks:** Security compliance. Unable to investigate unauthorized access.

**Current workaround:** None. No audit trail of API calls using credentials.

**Implementation:** Wire `src/core/auditLog.js` to integration handlers. Add audit entries for every API call using external credentials.

---

### Zenoti Appointment Webhooks Are Stubs

**Problem:** `src/core/integrationHandlers.js` has TODO comments. Zenoti events logged but not processed.

**Blocks:** Real-time availability sync. Currently manual sync only.

**Current workaround:** Managers manually trigger sync from UI.

**Implementation:** Implement appointment.booked, appointment.cancelled, appointment.modified handlers. Update availability cache and re-generate posts.

---

## Test Coverage Gaps

### messageRouter.js Has No Unit Tests

**What's not tested:** SMS/Telegram inbound routing, image processing, AI caption generation, draft approval flow, availability detection, Zenoti integration.

**Files:** `src/core/messageRouter.js` (1500+ lines, zero coverage)

**Risk:** Changes break silently. Only detected via live SMS testing (slow, manual, error-prone).

**Priority:** High — messageRouter is core business logic.

---

### Scheduler Retry Logic Not Tested

**What's not tested:** Retry exhaustion (MAX_RETRIES=3), error message translation, SMS notification to managers, state transitions.

**Files:** `src/scheduler.js:536–580`

**Risk:** Failed post handling is fragile. Untested edge cases break manager visibility.

**Priority:** High — users see failed posts.

---

### Zenoti Integration Endpoints Untested

**What's not tested:** Appointment fetching, working hours calculation, category inference, stylist fairness, availability image generation.

**Files:** `src/core/zenoti.js`, `src/core/zenotiSync.js`, `src/core/zenotiAvailability.js`, `src/routes/integrations.js`

**Risk:** Pro plan feature. Untested sync = customer escalations for missing slots.

**Priority:** High — critical for enterprise customers.

---

### No Tests for Multi-Location Switching

**What's not tested:** Session state after switch, IDOR checks (salon A accessing salon B), analytics scoping after switch.

**Files:** `src/routes/locations.js`, `src/middleware/tenantFromLink.js`, all manager routes

**Risk:** Manager from salon A could access salon B's data if tenant isolation breaks.

**Priority:** Critical — multi-tenancy security issue.

---

### No Tests for Image Generation Edge Cases

**What's not tested:** SVG injection via salon name, font loading failures, Puppeteer timeout handling, missing stock photos fallback chain.

**Files:** `src/core/buildPromotionImage.js`, `src/core/buildAvailabilityImage.js`, `src/core/puppeteerRenderer.js`

**Risk:** Image generation is critical path. Untested failures break availability/promotion posts.

**Priority:** High — visible user impact.

---

*Concerns audit: 2026-03-19*
