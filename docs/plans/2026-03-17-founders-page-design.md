# Founders Page — Design Doc
_Date: 2026-03-17_

## Summary

Two-page strategy: keep `index.html` as the evergreen 7-day trial page; add `/founders.html` as a dedicated ad landing page for the Founder offer. Founders get a 30-day trial + $50/month off for life via promo code `FOUNDER50`.

---

## Pages

### `index.html` (existing)
- Minimal changes — verify all copy says "7-day free trial" consistently
- CTA links: `https://app.mostlypostly.com/manager/signup` (no param)

### `founders.html` (new)
- Dedicated Founder landing page — single job: get the signup click
- CTA links: `https://app.mostlypostly.com/manager/signup?offer=founder`
- Sections:
  1. **Nav** — same as site-wide header
  2. **Hero** — eyebrow "Founding Member Offer", headline "Be one of the first. Pay less. Forever.", offer pills (30-Day Free Trial, $50 Off For Life), CTA button, promo code callout
  3. **How It Works** — 3 steps: sign up → trial → enter FOUNDER50 at checkout
  4. **Savings Block** — table showing regular vs founder rate per plan
  5. **Footer CTA** — repeat offer + button for bottom-of-page converts
  6. **Footer** — standard site footer

---

## App Change Required (`mostlypostly-app`)

**File:** `src/routes/billing.js` (checkout route)

**Change:** Read `?offer=founder` query param from the signup/checkout flow and pass `trial_period_days: 30` to Stripe instead of the default 7.

```js
const trialDays = req.query.offer === 'founder' ? 30 : 7;
// pass trialDays to Stripe checkout session creation
```

The `trial_used` flag already prevents double trials on the same account. Redemption limit on `FOUNDER50` in Stripe (set to max 10) prevents abuse.

---

## Promo Code

- Code: `FOUNDER50`
- Effect: $50 off first paid month (entered by user at Stripe checkout)
- Trial: 30 days free (handled by `?offer=founder` param in app)
- Stripe: Set redemption limit to match target founder count (e.g. 10)

---

## Out of Scope

- No countdown timers or fake urgency
- No founder-specific dashboard or onboarding changes
- Testimonial slots reserved but empty until real quotes collected

---

## Success Criteria

- [ ] `/founders.html` live on production site
- [ ] CTA passes `?offer=founder` to app signup URL
- [ ] App gives 30-day trial when param present, 7-day otherwise
- [ ] `index.html` consistently shows 7-day trial messaging
- [ ] `FOUNDER50` redemption limit set in Stripe Dashboard
