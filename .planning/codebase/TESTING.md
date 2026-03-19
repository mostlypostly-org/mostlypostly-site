# Testing Patterns

**Analysis Date:** 2026-03-19

## Test Framework

**Runner:** Vitest 3.2.4
- Config: None (uses defaults)
- Package.json: `"test": "vitest run"`
- Dev dependency only; not used in production

**Assertion Library:** Vitest built-in (standard Jest-compatible assertions)
- `expect()` for all assertions
- No additional assertion library needed

**Run Commands:**
```bash
npm test                    # Run all tests once
npm run test:watch         # Watch mode (if configured)
```

## Test File Organization

**Location:** `tests/` directory at project root
- Path: `/Users/troyhardister/chairlyos/mostlypostly/mostlypostly-app/tests/`
- Co-located tests: NO — tests are in a separate `tests/` directory, not alongside source files

**Naming Convention:**
- `{feature}.test.js` (e.g., `composeFinalCaption.test.js`, `vendorHashtags.test.js`)
- Test file imports functions from `src/` with relative paths

**Current Test Files:**
- `tests/composeFinalCaption.test.js` — Caption composition behavior (60 lines)
- `tests/vendorHashtags.test.js` — Hashtag normalization and building (68 lines)
- `tests/rcs.test.js` — RCS (Rich Communication Services) messaging (68 lines)
- `tests/igCollaborator.test.js` — Instagram collaborator tagging (30 lines)

## Test Structure

**Suite Organization** (Vitest/Jest standard):
```javascript
import { describe, it, expect } from "vitest";
import { composeFinalCaption } from "../src/core/composeFinalCaption.js";

describe("composeFinalCaption — Book CTA behavior", () => {
  it("IG with booking_url includes Book via link in bio", () => {
    const result = composeFinalCaption({
      caption: "Beautiful balayage",
      platform: "instagram",
      bookingUrl: "https://example.com/book",
    });
    expect(result).toContain("Book via link in bio.");
  });

  it("IG without booking_url does NOT include Book via link in bio", () => {
    const result = composeFinalCaption({
      caption: "Beautiful balayage",
      platform: "instagram",
      bookingUrl: "",
    });
    expect(result).not.toContain("Book via link in bio.");
  });
});
```

**Patterns Observed:**

1. **Shared Base Objects:** Tests use a base object for common test data, reducing duplication:
```javascript
const base = {
  caption: "Beautiful balayage",
  stylistName: "Jane",
  hashtags: [],
  salon: {},
};

describe("composeFinalCaption", () => {
  it("test 1", () => {
    const result = composeFinalCaption({ ...base, platform: "instagram" });
  });
  it("test 2", () => {
    const result = composeFinalCaption({ ...base, platform: "facebook" });
  });
});
```

2. **Focused Test Descriptions:** Each test describes a specific behavior, not implementation details:
   - ✅ Good: `"returns handle in array when ig_collab=1 and handle exists"`
   - ❌ Bad: `"calls array constructor and returns"`

3. **Arrangement-Act-Assert:** Three-phase test structure (implicit in all examples):
```javascript
// Arrange
const stylist = { ig_collab: 1, instagram_handle: "janedoe" };

// Act
const result = buildCollaborators(stylist);

// Assert
expect(result).toEqual(["janedoe"]);
```

## Mocking

**Framework:** Vitest built-in (`vi` module)
- `vi.fn()` — creates a mock function
- `vi.mock()` — mocks an entire module
- `vi.resetModules()` — resets module cache between tests

**Mocking Pattern: Module Mocks (RCS Test Example)**
```javascript
vi.mock("twilio", () => {
  const create = vi.fn().mockResolvedValue({ sid: "SM123" });
  const MessagingResponse = vi.fn(() => ({
    toString: () => "<Response/>",
    message: vi.fn(),
  }));
  const twilioConstructor = vi.fn(() => ({ messages: { create } }));
  twilioConstructor.twiml = { MessagingResponse };
  twilioConstructor.validateRequest = vi.fn(() => true);
  return {
    default: twilioConstructor,
    twiml: { MessagingResponse },
    validateRequest: vi.fn(() => true),
  };
});

// Must set env vars BEFORE importing the module under test
process.env.TWILIO_ACCOUNT_SID = "ACtest";
process.env.TWILIO_AUTH_TOKEN = "authtest";

describe("sendViaRcs", () => {
  let sendViaRcs;
  let mockCreate;

  beforeEach(async () => {
    vi.resetModules();
    process.env.RCS_ENABLED = "true";
    const twilio = await import("twilio");
    mockCreate = twilio.default().messages.create;
    const mod = await import("../src/routes/twilio.js");
    sendViaRcs = mod.sendViaRcs;
  });

  it("sends with persistentAction when RCS_ENABLED=true", async () => {
    await sendViaRcs("+15550001111", "Hello!", ["reply:Approve"]);
    expect(mockCreate).toHaveBeenCalledWith(
      expect.objectContaining({
        persistentAction: ["reply:Approve"],
        body: "Hello!",
      })
    );
  });
});
```

**Key Points from RCS Test:**
- Mocks are defined at module level with `vi.mock()`
- Environment variables are set before importing the module being tested
- `beforeEach()` resets modules and re-imports fresh instances
- Mock verification uses `toHaveBeenCalledWith()` with `expect.objectContaining()`

**What to Mock:**
- External API clients (Twilio, Twitter, etc.)
- Database calls (if testing functions that interact with DB)
- Environment-dependent behavior

**What NOT to Mock:**
- Internal utility functions (keep testing the full integration)
- Built-in functions (`String()`, `JSON.parse()`, etc.)
- Pure functions without side effects (test them directly)

## Fixtures and Factories

**Test Data Pattern:** Base object spread in test cases:
```javascript
const base = {
  caption: "Beautiful balayage",
  stylistName: "Jane",
  hashtags: [],
  salon: {},
};

// In each test:
composeFinalCaption({ ...base, platform: "instagram", bookingUrl: "..." })
```

**Location:**
- Inline in test files (no separate fixtures directory)
- Small, focused test data objects created per suite

**Data Builder Pattern (Hashtag Tests):**
```javascript
describe("buildVendorHashtagBlock", () => {
  it("takes first 3 salon tags, 2 brand tags, 1 product tag", () => {
    const result = buildVendorHashtagBlock({
      salonHashtags: ["#Hair", "#Style", "#Color", "#Extra"],
      brandHashtags: ["#AvedaColor", "#FullSpectrum"],
      productHashtag: "#Botanique",
    });
    expect(result).toBe("#Hair #Style #Color #AvedaColor #FullSpectrum #Botanique #MostlyPostly");
  });
});
```

## Coverage

**Requirements:** No coverage thresholds enforced
- No `.nycrc`, `coverage` config, or CI checks for coverage %
- Tests are added on-demand for critical business logic

**View Coverage** (if needed):
```bash
# Vitest doesn't have built-in coverage; would require:
npm install --save-dev @vitest/coverage-v8
vitest --coverage
```

## Test Types

**Unit Tests:**
- Scope: Single function with mocked dependencies
- Approach: Test inputs and outputs without side effects
- Example: `composeFinalCaption()` with various platform/hashtag combinations
- File: `tests/composeFinalCaption.test.js`

**Integration Tests (Implicit):**
- Scope: Multiple functions together (not yet formalized in test suite)
- Approach: Would test scheduler + publishers + DB together
- Current Gap: No integration tests present

**E2E Tests:**
- Status: Not used
- Reason: Manual testing on staging/production instead
- Alternative: Facebook/Instagram integration tested via `src/routes/analytics.js` debug endpoints

## Common Patterns

**Testing Array Outputs:**
```javascript
it("returns handle in array when ig_collab=1 and handle exists", () => {
  const stylist = { ig_collab: 1, instagram_handle: "janedoe" };
  expect(buildCollaborators(stylist)).toEqual(["janedoe"]);
});

it("strips leading @ from handle", () => {
  const stylist = { ig_collab: 1, instagram_handle: "@janedoe" };
  expect(buildCollaborators(stylist)[0]).toBe("janedoe");
});
```

**Testing Undefined Returns:**
```javascript
it("returns undefined when ig_collab=0", () => {
  const stylist = { ig_collab: 0, instagram_handle: "janedoe" };
  expect(buildCollaborators(stylist)).toBeUndefined();
});
```

**Testing String Containment:**
```javascript
it("IG with booking_url includes Book via link in bio", () => {
  const result = composeFinalCaption({
    caption: "Beautiful balayage",
    platform: "instagram",
    bookingUrl: "https://example.com/book",
  });
  expect(result).toContain("Book via link in bio.");
});

it("Facebook with booking_url includes the URL", () => {
  const result = composeFinalCaption({
    caption: "Beautiful balayage",
    platform: "facebook",
    bookingUrl: "https://example.com/book",
  });
  expect(result).toContain("https://example.com/book");
});
```

**Testing Negative Cases (NOT Contains):**
```javascript
it("IG without booking_url does NOT include Book via link in bio", () => {
  const result = composeFinalCaption({
    caption: "Beautiful balayage",
    platform: "instagram",
    bookingUrl: "",
  });
  expect(result).not.toContain("Book via link in bio.");
  expect(result).not.toContain("Book now");
});
```

**Testing with Async/Mocks:**
```javascript
it("sends with persistentAction when RCS_ENABLED=true", async () => {
  await sendViaRcs("+15550001111", "Hello!", ["reply:Approve", "reply:Cancel"]);
  expect(mockCreate).toHaveBeenCalledWith(
    expect.objectContaining({
      persistentAction: ["reply:Approve", "reply:Cancel"],
      body: "Hello!",
      to: "+15550001111",
    })
  );
});
```

## Test Execution

**Running Tests:**
```bash
npm test                    # Runs vitest run (once, exits)
npm run test:watch         # Recommended for development (not configured in package.json yet)
```

**Test Output:**
- Vitest prints pass/fail for each test
- Failed assertions show expected vs actual
- Error messages from thrown exceptions appear in output

## Testing Gaps

**Not Currently Tested:**
1. **Database interactions** — `storage.js`, `salonLookup.js` (DB operations without mocks)
2. **Scheduler logic** — `scheduler.js` (complex scheduling algorithms)
3. **Message routing** — `messageRouter.js` (complex multi-step workflow)
4. **API integrations** — Facebook, Instagram, Google Business Profile publish flows (would need mocks)
5. **Image generation** — `buildAvailabilityImage.js`, `buildPromotionImage.js` (SVG/image output)
6. **Zenoti integration** — `zenotiSync.js`, `zenoti.js` (requires Zenoti API mocks)

**Priority for New Tests:**
1. **High:** Scheduler priority logic, post composition, error translation
2. **Medium:** Availability calculation, vendor hashtag building
3. **Low:** Image generation (visual testing required anyway)

## Best Practices for Adding Tests

1. **Test one thing per test:** Each `it()` should verify one behavior
2. **Use descriptive test names:** Readers should understand what's tested without reading code
3. **Keep tests small and fast:** <100ms per test if possible
4. **Mock external dependencies:** Don't make real API calls or DB writes during tests
5. **Reuse test data:** Use base objects to reduce duplication
6. **Test edge cases:** Empty inputs, null values, falsy vs nullish, boundary conditions
7. **Avoid testing implementation:** Test behavior and outputs, not internal structure

## Debugging Tests

**Print Debugging:**
```javascript
it("test something", () => {
  const result = myFunction({ some: "data" });
  console.log("Result:", result);  // Will print to console
  expect(result).toBe("expected");
});
```

**Running a Single Test:**
```bash
npx vitest run tests/composeFinalCaption.test.js  # Run one file
npx vitest run tests/composeFinalCaption.test.js -t "IG with booking"  # Run tests matching pattern
```

**Skipping Tests:**
```javascript
describe.skip("These are skipped", () => {
  it("won't run", () => {});
});

it.skip("this test is skipped", () => {});
```

**Running Only One Test:**
```javascript
it.only("Run only this test", () => {});
```

---

*Testing analysis: 2026-03-19*
