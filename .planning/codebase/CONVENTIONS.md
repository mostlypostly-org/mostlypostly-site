# Coding Conventions

**Analysis Date:** 2026-03-19

## Module System

**Type:** ES Modules (ESM) throughout the codebase
- Always use `import`/`export` statements
- Never use `require()` or `module.exports`
- File extension: `.js` for all modules
- Package.json specifies `"type": "module"`

**Example:**
```javascript
import { db } from "../../db.js";
import { publishToFacebook } from "../publishers/facebook.js";
export function composeFinalCaption({ caption, hashtags, platform }) { }
export async function syncSalonInsights(salonId) { }
```

## Naming Patterns

**Files:**
- `camelCase.js` for all filenames (e.g., `messageRouter.js`, `salonLookup.js`)
- Descriptive, action-oriented names (e.g., `publishToFacebook.js`, `buildAvailabilityImage.js`)
- Grouping by feature: route files in `src/routes/`, core logic in `src/core/`, publishers in `src/publishers/`, utilities in `src/utils/`

**Functions:**
- `camelCase` for function names
- Action-based verbs: `get*`, `fetch*`, `build*`, `publish*`, `save*`, `sync*`
- Examples: `getSalonById()`, `fetchPexelsBackground()`, `buildVendorHashtagBlock()`, `syncSalonInsights()`
- Private/internal helpers: optionally prefixed with `_` (e.g., `_mergeHashtags()`)

**Variables:**
- `camelCase` for all variables and constants
- `UPPER_CASE` for environment-derived or configuration constants that are not reassigned
  - Example: `const BASE_URL = () => (process.env.PUBLIC_BASE_URL || 'https://...')` (function) vs `const BRAND_TAG = process.env.MOSTLYPOSTLY_BRAND_TAG || "#MostlyPostly"` (immutable)
  - Example: `const PLAN_LIMITS = { ... }` or `const FEED_TYPES = new Set([...])`
- Descriptive names: `salonId` not `sid`, `managerPhone` not `mgrPhone`

**Types:**
- Salon IDs: always stored as strings (slug format, e.g., `"vanity-lounge"`)
- UUIDs: generated with `crypto.randomUUID()`
- Booleans: 0/1 in database (SQLite), actual boolean in JavaScript
- Time: ISO 8601 strings for UTC (`new Date().toISOString()`), Luxon DateTime for timezone-aware operations

**SQL Identifiers:**
- Table names: `snake_case` (e.g., `salon_vendor_approvals`, `post_insights`)
- Column names: `snake_case` (e.g., `facebook_page_id`, `created_at`)
- Prepared statements use `?` placeholders or named `@param` placeholders

## Code Style

**Formatting:**
- No automated linter (ESLint) or formatter (Prettier) configured
- Indentation: 2 spaces (observed in all code)
- Line length: no hard limit enforced; readability-focused
- Semicolons: always used at end of statements
- String quotes: double quotes `"` preferred (observed in codebase)

**Spacing:**
- Space after `function`, `if`, `for`, etc.
- Space around binary operators: `a + b`, `x === y`
- Blank lines between logical sections within a function/block

**Example:**
```javascript
function withinPostingWindow(now, start, end) {
  const [sH, sM] = start.split(":").map(Number);
  const [eH, eM] = end.split(":").map(Number);
  const windowStart = now.set({ hour: sH, minute: sM, second: 0, millisecond: 0 });
  const windowEnd = now.set({ hour: eH, minute: eM, second: 59, millisecond: 999 });
  return now >= windowStart && now <= windowEnd;
}
```

**Linting:**
- No ESLint configuration present
- No automated style enforcement during development
- Style is maintained through code review and CLAUDE.md conventions

## Import Organization

**Order** (observed pattern):
1. Node.js built-in modules (`crypto`, `fs`, `path`, `http`, etc.)
2. Third-party packages (`express`, `better-sqlite3`, `luxon`, `dotenv`, etc.)
3. Local relative imports (`../../db.js`, `../utils/*.js`, etc.)
4. Blank line between groups

**Example from `src/scheduler.js`:**
```javascript
import { DateTime } from "luxon";
import { db } from "../db.js";
import { createLogger } from "./utils/logHelper.js";
import { publishToFacebook, publishToFacebookMulti } from "./publishers/facebook.js";
import { publishToInstagram } from "./publishers/instagram.js";
```

**Path Resolution:**
- All imports use relative paths with explicit `./` or `../` (no path aliases)
- Paths are always correct relative to the importing file's location
- Example: `src/routes/manager.js` imports `../../db.js` (2 levels up to project root)

## Error Handling

**Pattern: Try/Catch with Specific Error Messages**
```javascript
try {
  const u = new URL(url);
  u.searchParams.set('utm_source', source);
  return u.toString();
} catch {
  return url; // Graceful fallback
}
```

**Pattern: Throw with Context**
```javascript
if (!token) {
  throw new Error("Missing Facebook access token (no salon token and no env token)");
}

if (!pageId || typeof pageId !== "string") {
  console.error("❌ [Facebook] Invalid pageId:", pageOrSalon);
  throw new Error("Facebook publisher received invalid pageId");
}
```

**Pattern: Error Translation (User-Facing)**
- `src/core/postErrorTranslator.js` translates raw API errors to plain English
- Uses regex rules to match error patterns and return user-friendly messages
- Fallback: generic message if no rule matches

```javascript
const RULES = [
  { match: /pages_read_engagement|token.*expir/i, text: "Facebook connection needs to be refreshed..." },
];

export function translatePostError(rawError) {
  for (const rule of RULES) {
    if (rule.match.test(rawError)) return rule.text;
  }
  return "Post failed to publish...";
}
```

**Pattern: Logging Errors**
- Console logs: `console.log()`, `console.warn()`, `console.error()`
- Use emoji prefixes for visual scanning
- Include context in messages

## Logging

**Framework:** `console` and file-based via `src/utils/logHelper.js`

**Structured Logging Pattern:**
```javascript
const log = createLogger("scheduler");
log("post_published", { salon_id: "venue-name", post_id: 42, platform: "instagram" });
```

**Log Locations:**
- File logs: `data/logs/{name}.log` (JSON lines format)
- Console output: includes timestamp, UUID, and event data

**Console Emoji Convention:**
- ✅ Success/completion
- ❌ Error/failure
- ⚠️ Warning
- ℹ️ Information
- 🚀 Starting/launching
- 📤 Sending/uploading
- 🧠 Logic/decision point
- ⏱ Timing measurement

**Example:**
```javascript
console.log("✅ [Facebook] Photo post success:", data);
console.warn("⚠️ [Facebook] Photo upload failed:", msg);
console.log(`⏱ ${label}: ${elapsed}ms`);
```

## Comments

**When to Comment:**
- Complex business logic that isn't obvious from code
- Workarounds or hacks (include why if non-obvious)
- Significant algorithm explanation
- Avoid: Comments explaining what obviously-named code does

**JSDoc Style:**
- Used for exported functions and significant parameters
- Single-line format for simple functions:

```javascript
/**
 * slugify("Jessica M") → "jessica-m"
 */
export function slugify(str) { }
```

- Multi-line for complex signatures:

```javascript
/**
 * Compose a final caption string with consistent, readable formatting.
 * - Merges AI + salon hashtags + brand tag (deduped)
 * - Handles both text and HTML modes
 */
export function composeFinalCaption({ caption, hashtags, platform }) { }
```

**Inline Comments:**
- Minimal use; prefer clear code
- When used: explain WHY, not WHAT
- Example: `// Fallback to text-only post if image upload fails`

## Function Design

**Size:** Keep functions under 100 lines; break complex operations into smaller functions

**Parameters:**
- Use destructuring for object parameters (common pattern in this codebase)
- Named parameters are preferred over positional

**Example:**
```javascript
export function composeFinalCaption({
  caption,
  hashtags = [],
  platform = "generic",
  salonId = null,
  postId = null,
}) { }
```

**Return Values:**
- Explicit returns, no implicit undefined
- Return early on error conditions:

```javascript
if (!salonSlug) return null;
if (!row) return null;
return row;
```

**Async Functions:**
- Always wrap in try/catch (not just `.catch()` on promises)
- Make async intent clear at declaration: `async function` or `async (args) => {}`

## Module Design

**Exports:**
- Prefer named exports over default exports
- All public functions are explicitly exported

```javascript
export function buildCollaborators(stylist) { }
export function normalizeHashtag(tag) { }
export function buildVendorHashtagBlock({ salonHashtags, brandHashtags }) { }
```

**Barrel Files:**
- Not used; imports are direct to the source file
- Example: `import { buildPromotionImage } from "../core/buildPromotionImage.js"`

**Single Responsibility:**
- Each file has a clear focus
- Related helpers co-locate in one file (e.g., `normalizeHashtag()` and `buildVendorHashtagBlock()` in `vendorScheduler.js`)

## Database Interactions

**Pattern: Prepared Statements**
- Always use parameterized queries to prevent SQL injection
- Named parameters (`@name`) or positional (`?`)

```javascript
const stmt = db.prepare("SELECT * FROM salons WHERE slug = ?");
const row = stmt.get(salonSlug);

db.prepare("INSERT INTO posts (id, salon_id, ...) VALUES (@id, @salon_id, ...)").run({
  id: crypto.randomUUID(),
  salon_id: salonId,
});
```

**Pattern: Synchronous DB Access**
- `better-sqlite3` is synchronous; all DB calls are blocking
- No `await` on DB statements

**Row Naming:**
- Single row result: `const row = db.prepare(...).get(...)`
- Array of rows: `db.prepare(...).all(...)`
- Counts: `const { n } = db.prepare("SELECT COUNT(*) as n ...").get(...)`

## Type Coercion & Validation

**String Coercion:**
- Use `String(value)` or `(value || "").toString()` when unsure
- Always trim: `.trim()`
- Example: `const slug = String(salonSlug).trim().toLowerCase();`

**Optional Chaining & Nullish Coalescing:**
- Use `?.` for safe property access on potentially null objects
- Use `??` for nullish coalescing (not falsy)
- Example: `salon?.booking_url || ""`

**Boolean Handling (Database):**
- Database stores 0/1 (SQLite INTEGER)
- JavaScript: use loose comparison or explicit boolean
- Example: `if (stylist.ig_collab)` reads 0/1 as falsy/truthy

## Sensitive Data Handling

**Secrets & Credentials:**
- Never log, console.log, or expose in error responses
- Never store in codebase (use `.env` with `dotenv`)

**Encryption:**
- Sensitive values like Zenoti API keys are encrypted at rest using `src/core/encryption.js`
- IP addresses in analytics: always SHA-256 hashed, never stored raw
- Example: `utm_clicks.ip_hash` stores `SHA-256(visitor_ip)`

## Configuration

**Environment Variables:**
- Loaded from `.env` via `dotenv` package
- Access via `process.env.VAR_NAME`
- Defaults embedded in code when appropriate:

```javascript
const APP_ENV = process.env.APP_ENV || process.env.NODE_ENV || "local";
const BASE_URL = () => (process.env.PUBLIC_BASE_URL || 'https://app.mostlypostly.com').replace(/\/$/, '');
```

**Constants:**
- Configuration constants at top of each file
- Example from `scheduler.js`:

```javascript
const FORCE_POST_NOW = process.env.FORCE_POST_NOW === "1";
export const DEFAULT_PRIORITY = [ "availability", "before_after", ... ];
const FEED_TYPES = new Set([ "standard_post", "before_after", ... ]);
```

---

*Convention analysis: 2026-03-19*
