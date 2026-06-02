# RFC: New Pricing Tiers — Standard, Pro Monthly, and Pro Annual

**Metadata:**
- **Author:** Bucks Team
- **Status:** Draft
- **Reviewers:** CPO, Product Owner, Engineering Lead
- **Type:** #feature-rfc
- **Created:** June 2026

---

## 1. Problem

Bucks Currency Converter currently has a single paid tier at **$7.99/month** (legacy `bucks_premium`). This one-size-fits-all model creates several friction points:

**Merchant pain points:**
- No incentive for annual commitment — legacy annual plans required manual coupon-based discounting, which was ad-hoc and inconsistent.
- Power merchants who need analytics have no upgrade path; analytics is gated but there's no plan to unlock it.
- Price-sensitive merchants who want "just the widget" are paying the same as power users who expect deeper features.

**Business gaps:**
- No monetization of the analytics feature despite significant investment in building it.
- No recurring annual revenue stream for new users (only legacy subscribers have annual plans).
- Single pricing tier makes it impossible to test conversion at different price points.

**Current workarounds:**
- Annual discounts issued manually via Shopify partner coupon codes (`bucks_premium_60`, `_65`, `_70`).
- Analytics page is accessible for all paid plans by default — there's no gate to drive upgrades.

**Competitor context:**
- Most Shopify currency/FX apps (e.g., BEST Currency Converter, Auto Currency Switcher) offer 2–3 pricing tiers with clear feature differentiation, making upsell paths obvious to merchants.

---

## 2. Why This Matters?

**Why now?**
- The analytics module is built and live but not yet monetized.
- Legacy annual plan management (manual coupon codes) does not scale and creates customer confusion.
- The existing $7.99 monthly plan is below market for the feature set we now offer.

**What we gain:**
- **Revenue uplift**: New Standard tier ($9.99/mo) replaces $7.99 for all new subscribers — a $2/month/merchant increase. Pro tier ($14.99/mo) creates a clear upgrade path for analytics-hungry merchants.
- **Annual predictability**: Pro Annual at $95.88/year (20% off $9.99 × 12) locks merchants in, reducing churn and improving LTV.
- **Analytics monetization**: Full analytics becomes a tangible Pro feature, giving free and Standard users a clear reason to upgrade.
- **Pricing clarity**: Replaces ad-hoc coupon-based annual discounts with a transparent, self-serve annual plan.

**Cost of NOT doing this:**
- Analytics investment continues to go unmonetized.
- Legacy annual plan coupon management grows harder as subscriber base scales.
- Revenue stays capped at $7.99/merchant/month for all new signups.

---

## 3. Proposed Solution

Introduce **3 new plans** while keeping legacy plans accessible only to their current subscribers:

| Plan | DB ID | Price | Analytics | Notes |
|---|---|---|---|---|
| Standard Monthly | `bucks_premium_standard` | $9.99/mo | ❌ None | No coupon; replaces $7.99 for new users |
| Pro Monthly | `bucks_premium_pro` | $14.99/mo | ✅ Full | Coupon `BUCKS33` → $9.99/mo |
| Pro Annual | `bucks_premium_pro_annual` | $95.88/yr | ✅ Full | 20% off ($9.99 × 12 × 0.80); no coupon needed |

**Annual price derivation:**
```
Base monthly (Standard): $9.99
Annual equivalent:        $9.99 × 12  = $119.88
20% annual discount:      $119.88 × 0.80 = $95.904 ≈ $95.88/year
Effective monthly:        $95.88 / 12  = $7.99/month
```

> Note: $95.88/year happens to equal the legacy annual full-price ($7.99 × 12), creating a natural price anchor — Pro Annual costs the same as the old plan but now includes full analytics.

**Coupon scope — BUCKS33:**
- Applies **only** to Pro Monthly ($14.99 → $9.99/month).
- Does **not** apply to Pro Annual — the 20% discount is already built into the $95.88 price.
- Does **not** apply to Standard Monthly ($9.99 is already the base price).

**Key design decisions:**

- **$9.99 is a separate plan** — not a coupon-applied $14.99. It has its own `bucks_plan` DB value (`bucks_premium_standard`) and is a distinct Shopify subscription.
- **Legacy plans are not deprecated** — existing $7.99 monthly and legacy annual subscribers keep their plans. They see upgrade options but are never force-migrated.
- **`replacementBehavior: "STANDARD"`** is set only for same-interval upgrades from an active paid plan, so Shopify pro-rates the remaining balance immediately.
- **Cross-interval upgrades** (e.g., monthly → annual) rely on Shopify's default behavior — consistent with how the current codebase already handles it.

**Edge cases:**
- A legacy $7.99 monthly subscriber clicking Pro Annual (cross-interval): no `replacementBehavior` → Shopify defaults apply, old monthly removed when annual activates.
- A legacy annual subscriber clicking Standard $9.99 (cross-interval): same as above.
- Trial days: same 7-day trial logic for all new plans; users who have exhausted their trial get 0 trial days.
- Coupon entered for Standard/Annual plan: backend ignores/rejects coupon; price unchanged.

---

## 4. End-to-End Flow

### 4a. Merchant Segment: New User / Existing Free

**Pricing page shows 4 cards:**
```
┌──────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐
│   Free   │  │  Standard      │  │  Pro Monthly   │  │   Pro Annual       │
│   $0/mo  │  │  $9.99/mo      │  │  $14.99/mo     │  │   $95.88/yr        │
│ [Current]│  │ [Start Trial]  │  │  [BUCKS33 →    │  │ [Start Trial]      │
│          │  │                │  │   $9.99/mo]    │  │ ($7.99/mo equiv)   │
└──────────┘  └────────────────┘  └────────────────┘  └────────────────────┘
```

**Merchant flow:**
1. Opens Pricing page → sees 4 cards
2. Clicks "Start Trial" on Standard → redirected to Shopify billing confirmation
3. Approves → Shopify creates subscription → `checkSubscriptionStatus` writes `bucks_plan: "bucks_premium_standard"` to DB
4. Redirected back to pricing page with success message

**Pro Monthly with coupon:**
1. Merchant clicks "Start Trial" on Pro Monthly card
2. Card shows "Use code BUCKS33 for $9.99/month" footer text
3. Merchant enters `BUCKS33` in coupon field → card shows struck-through $14.99, new price $9.99
4. Clicks upgrade → API call to `initSubscription` with `couponCode: "BUCKS33"`
5. Backend validates coupon → sets `price.amount = 9.99` before Shopify mutation
6. Shopify confirmation → approved → DB updated with `bucks_plan: "bucks_premium_pro"`

---

### 4b. Merchant Segment: Existing $7.99 Monthly

**Pricing page shows 5 cards:**
```
┌──────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐
│   Free   │  │  Legacy $7.99  │  │  Standard      │  │  Pro Monthly   │  │  Pro Annual        │
│   $0/mo  │  │  [Current Plan]│  │  $9.99/mo      │  │  $14.99/mo     │  │  $95.88/yr         │
│[Downgrade│  │                │  │  [Upgrade]     │  │  [BUCKS33 →    │  │  [Upgrade]         │
│  ]       │  │                │  │  ↑ STANDARD    │  │   $9.99/mo]    │  │  (Shopify default) │
└──────────┘  └────────────────┘  └────────────────┘  └──── STANDARD ─┘  └────────────────────┘
```

**Same-interval upgrade (Monthly → Standard $9.99 or Pro $14.99):**
1. Merchant clicks upgrade on Standard or Pro Monthly card
2. `initSubscription` detects: `currentPlan = "bucks_premium"` (monthly), `newPlan interval = monthly` → same interval
3. Sets `replacementBehavior: "STANDARD"` in Shopify mutation
4. Shopify: credits remaining $7.99 balance, removes old subscription immediately, creates new subscription
5. Merchant sees Shopify confirmation → approves → DB updated

**Cross-interval upgrade (Monthly → Pro Annual):**
1. Same flow but `replacementBehavior` is omitted
2. Shopify uses default transition behavior — old monthly removed when new annual activates

---

### 4c. Merchant Segment: Existing Annual Subscriber

**Pricing page shows 5 cards:**
```
┌──────────┐  ┌────────────────┐  ┌────────────────┐  ┌─────────────────────┐  ┌────────────────────┐
│   Free   │  │  Standard      │  │  Pro Monthly   │  │  Legacy Annual      │  │  Pro Annual        │
│   $0/mo  │  │  $9.99/mo      │  │  $14.99/mo     │  │  [Current Plan]     │  │  $95.88/yr         │
│[Downgrade│  │  [Upgrade]     │  │  [Upgrade]     │  │                     │  │  [Upgrade]         │
│  ]       │  │ (Shopify deflt)│  │ (Shopify deflt)│  │                     │  │  ↑ STANDARD        │
└──────────┘  └────────────────┘  └────────────────┘  └─────────────────────┘  └────────────────────┘
```

- **Annual → Pro Annual** (same interval): `replacementBehavior: "STANDARD"` → remaining annual balance credited
- **Annual → Standard/Pro Monthly** (cross-interval): Shopify default; existing annual continues until cycle end or is replaced when new monthly activates

---

### 4d. Technical Flow — `initSubscription.js`

```
POST /api/v1/billing/initSubscription
  │
  ├─ Fetch userRecord from DB
  │    - bucks_plan, plan_type, subscription_id
  │
  ├─ Determine replacementBehavior
  │    - isOnActivePaidPlan = isPremiumPlan(bucks_plan) && subscription_id exists
  │    - currentInterval = getPlanType(userRecord)  // "monthly" | "annual"
  │    - newInterval = body.plan_type
  │    - isSameInterval = currentInterval === newInterval
  │    - replacementBehavior = (isOnActivePaidPlan && isSameInterval) ? "STANDARD" : undefined
  │
  ├─ Validate coupon (Pro Monthly only)
  │    - if bucks_plan == "bucks_premium_pro" && couponCode === "BUCKS33"
  │        → override price.amount = 9.99
  │    - if bucks_plan == "bucks_premium_pro_annual" OR "bucks_premium_standard"
  │        → ignore couponCode entirely
  │
  ├─ Determine trial days (existing logic unchanged)
  │
  └─ Call createAppSubscription(session, subscriptionDetails, shop, testMode, trialDays, replacementBehavior)
       │
       └─ Shopify mutation → returns confirmationUrl
            │
            └─ Redirect merchant to Shopify billing confirmation
                 │
                 └─ On approval → checkSubscriptionStatus
                      → write bucks_plan, plan_type, subscription_id to DB
```

---

## 5. Implementation Details

### Data Model
No new database columns required. The existing `users` table fields handle all new plans:
- `bucks_plan` (String) — stores new values: `bucks_premium_standard`, `bucks_premium_pro`, `bucks_premium_pro_annual`
- `plan_type` (String) — `"monthly"` or `"annual"` (unchanged)
- `subscription_id` (String) — Shopify subscription GID (unchanged)

### Backend Changes

**`utils/graphQl/createAppSubscription.js`**
- Add `replacementBehavior` as a new parameter
- Include it conditionally in the GraphQL mutation variables (only when not `undefined`)

```graphql
# Updated mutation signature
mutation AppSubscriptionCreate(
  $name: String!
  $lineItems: [AppSubscriptionLineItemInput!]!
  $returnUrl: URL!
  $trialDays: Int!
  $replacementBehavior: AppSubscriptionReplacementBehavior  # NEW (optional)
) {
  appSubscriptionCreate(
    name: $name
    returnUrl: $returnUrl
    lineItems: $lineItems
    test: false
    trialDays: $trialDays
    replacementBehavior: $replacementBehavior  # passed only when STANDARD
  ) { ... }
}
```

**`pages/api/v1/billing/initSubscription.js`**
- Add same-interval detection logic for `replacementBehavior`
- Add coupon validation scoped to `bucks_premium_pro` only
- Pass `replacementBehavior` through to `createAppSubscription`

**`utils/common/common-methods/pricingConfig.js` — `getPlanType`**
- Add cases for `bucks_premium_standard` (returns `"monthly"`), `bucks_premium_pro` (returns `"monthly"`), `bucks_premium_pro_annual` (returns `"annual"`)

**`pages/api/v1/billing/checkSubscriptionStatus.js`**
- No changes needed — already reads `bucks_plan` and `plan_type` from query params and writes to DB

### Frontend Changes

**`utils/constants.js` — `PREMIUM_PLANS`**
```js
export const PREMIUM_PLANS = {
  MONTHLY: "bucks_premium",
  ANNUAL_60: "bucks_premium_60",
  ANNUAL_65: "bucks_premium_65",
  ANNUAL_70: "bucks_premium_70",
  STANDARD_MONTHLY: "bucks_premium_standard",  // NEW
  PRO_MONTHLY: "bucks_premium_pro",            // NEW
  PRO_ANNUAL: "bucks_premium_pro_annual",      // NEW
};
```

**`utils/common/common-methods/trialHelpers.js` — `PREMIUM_PLAN_IDS`**
- Add `"bucks_premium_standard"`, `"bucks_premium_pro"`, `"bucks_premium_pro_annual"`

**`utils/common/constants/constants.js` — new pricing constants & `planDetails` entries**
```js
export const NEW_STANDARD_MONTHLY_PRICE = 9.99;
export const NEW_PRO_MONTHLY_PRICE = 14.99;
export const NEW_PRO_ANNUAL_PRICE = 95.88;           // $9.99 × 12 × 0.80
export const NEW_PRO_ANNUAL_FULL_PRICE = 119.88;     // $9.99 × 12 (compare price for strikethrough)
export const PRO_COUPON_CODE = "BUCKS33";             // Pro Monthly only
export const NEW_PRO_MONTHLY_PRICE_WITH_COUPON = 9.99;
```

Three new `planDetails` entries with IDs: `"standard"`, `"pro"`, `"pro-annual"`.

**`pages/pricing/index.jsx`**
- Build `visiblePlanIds` array per user segment:
  - Free/New: `["free", "standard", "pro", "pro-annual"]` — 4 cards
  - Legacy Monthly: `["free", "legacy-monthly", "standard", "pro", "pro-annual"]` — 5 cards
  - Legacy Annual: `["free", "standard", "pro", "legacy-annual", "pro-annual"]` — 5 cards
- Dynamic `InlineGrid` columns: `mdDown ? 1 : lgDown ? 2 : cardCount`

**`components/pricing/PricingCard.jsx`**
- Add coupon UI (input field or inline display) — rendered only when `plan.couponCode` exists
- Show struck-through original price and discounted price when coupon applied
- `isCurrentPlanCard` check extended to all new plan IDs

**`pages/analytics/index.jsx`**
```js
const PRO_ANALYTICS_PLANS = ["bucks_premium_pro", "bucks_premium_pro_annual"];
const hasFullAnalytics = PRO_ANALYTICS_PLANS.includes(userData?.bucks_plan);
// If !hasFullAnalytics → show teaser/upgrade prompt
```

### PostHog Events
| Event | Trigger |
|---|---|
| `"bucks standard plan initiated"` | Standard $9.99 selected |
| `"bucks pro plan initiated"` | Pro $14.99 or Pro Annual $95.88 selected |
| `"bucks pro coupon applied"` | BUCKS33 coupon applied (track final amount) |
| `"bucks pro plan selected"` | Shopify billing callback confirms new Pro plan |

---

## 6. Open Questions

- **Coupon expiry:** Is `BUCKS33` always active or time-limited? If time-limited, should expiry be in an env var?
- **Analytics teaser UX:** Blur overlay vs. locked rows vs. date-limited view? Needs design input.
- **5-card grid on tablet:** 2-column wrap (3 rows) or horizontal scroll? Confirm breakpoints.
- **Legacy plan CTA copy:** "Upgrade to Pro" vs "Switch to Pro" vs "Unlock Analytics"?
- **Standard plan display name:** "Standard Monthly", "Essential", or "Plus"?
- **Coupon UI placement:** Inline text under price with a copy button, or an explicit input field requiring manual entry?

---

## References

- [Shopify AppSubscriptionCreate mutation docs](https://shopify.dev/docs/api/admin-graphql/latest/mutations/appSubscriptionCreate)
- [Shopify AppSubscriptionReplacementBehavior enum](https://shopify.dev/docs/api/admin-graphql/latest/enums/AppSubscriptionReplacementBehavior)
- [Detailed plan specification: `docs/new-pricing-plans.md`](./new-pricing-plans.md)

---

## Implementation Plan

| Phase | Goal | Files Affected | Estimated Effort |
|---|---|---|---|
| **Phase 1** | Constants, PREMIUM_PLANS, PREMIUM_PLAN_IDS, planDetails, getPlanType | `constants.js`, `trialHelpers.js`, `pricingConfig.js` | 0.5 day |
| **Phase 2** | Backend: `createAppSubscription` + `initSubscription` (replacementBehavior + coupon logic) | `createAppSubscription.js`, `initSubscription.js` | 1 day |
| **Phase 3** | Frontend: Pricing page filtering, dynamic grid, coupon UI, PricingCard updates | `pages/pricing/index.jsx`, `PricingCard.jsx` | 1.5 days |
| **Phase 4** | Analytics gate: hasFullAnalytics check + teaser UI | `pages/analytics/index.jsx` | 0.5 day |
| **Phase 5** | QA: Test all 3 user segments, coupon flow, replacementBehavior paths | — | 1 day |

Each phase can be shipped independently. Phase 1 is a prerequisite for all others.
