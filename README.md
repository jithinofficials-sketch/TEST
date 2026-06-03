# RFC: New Pricing Tiers — Standard, Pro Monthly, and Pro Annual

**Metadata:**
- **Author:** Bucks Team
- **Status:** Draft
- **Reviewers:** CPO, Product Owner, Engineering Lead
- **Type:** #feature-rfc
- **Created:** June 2026

---

## 1. Problem

**Current state:**
- Bucks has a single paid tier at $7.99/month that hasn't changed since launch
- Legacy annual plans exist (`bucks_premium_60`, `_65`, `_70`) but are only for existing subscribers via manual coupon codes — new users have no self-serve annual option
- Analytics is fully built and available to all paid users, but there's no way to monetize it — no plan gates it

**Problems:**
1. **Revenue is capped at $7.99/month** — below market for the current feature set. New subscribers should be on a higher price point.
2. **Analytics investment is unmonetized** — we built it but have no paid tier that includes it as a differentiator.
3. **No self-serve annual plan for new users** — legacy annual plans are manual coupon-based and not available to new signups.

---

## 2. Why This Matters?

- **Price increase for new users**: New subscribers see a **$9.99/month** Standard plan (shown as "Plus Plan" on the pricing card) instead of the old $7.99.
- **Analytics upgrade incentive**: A **$19.99/month Pro plan** includes full analytics. Merchants apply coupon `BUCKS33` (50.03% off) to bring it to **$9.99/month** — same price as Standard but with analytics unlocked. This creates a clear, visible reason to upgrade.
- **Self-serve annual plan**: A **$95.90/year** Pro Annual plan (39.98% of $239.88 full price) replaces ad-hoc manual coupon-based annual discounts. Partners get an additional 5% off: **$83.93/year** (34.98% of $239.88).
- **$19.99 plan is temporary**: Once the upgrade behaviour is established, the $19.99/month plan will be removed. $9.99 Standard remains as the primary ongoing paid tier.

**If we don't do this:** Revenue stays at $7.99/month for all new signups, analytics goes unmonetized, and legacy annual coupon management continues to scale poorly.

---

## 3. Proposed Solution

Introduce **3 new plans** while keeping legacy plans accessible only to their current subscribers:

| Plan | DB ID | Price | Analytics | Notes |
|---|---|---|---|---|
| Standard Monthly | `bucks_premium_standard` | $9.99/mo | ❌ None | No coupon; for new/free users only (shown as "Plus Plan") |
| Pro Monthly | `bucks_premium_pro` | $19.99/mo | ✅ Full | Coupon `BUCKS33` (50.03% off) → $9.99/mo; for existing paid users |
| Pro Annual | `bucks_premium_pro_annual` | $95.90/yr | ✅ Full | 39.98% of $239.88 full price; no coupon; partners pay $83.93/yr |

**Annual price derivation:**
```
Pro Monthly full price:   $19.99/mo
Annual equivalent:        $19.99 × 12 = $239.88 (compare/strikethrough price)
Default annual price:     $239.88 × 0.3998 = $95.90/year  (merchant pays 39.98% of full)
Partner annual price:     $239.88 × 0.3498 = $83.93/year  (merchant pays 34.98% of full)
Effective monthly:        $95.90 / 12 = $7.99/month
```

> The $95.90/year effective monthly rate ($7.99) matches the legacy plan price — a natural anchor showing Pro Annual delivers analytics at no extra monthly cost vs. the old plan.

**Coupon scope — BUCKS33:**
- Applies **only** to Pro Monthly ($19.99 → $9.99/month, 50.03% off).
- Does **not** apply to Pro Annual — discount is built into the $95.90 price.
- Does **not** apply to Standard Monthly — $9.99 is a fixed base price, no coupon accepted.

**Key design decisions:**

- **Standard ($9.99) and Pro ($19.99) are separate plans** — each has its own `bucks_plan` DB value and distinct Shopify subscription. $9.99 is not a coupon-applied $19.99.
- **Standard is for new/free users only** — existing paid users never see the $9.99 Standard card. Their upgrade path is directly to Pro $19.99 or Annual $95.90.
- **Legacy cards hide after upgrade** — once an existing monthly subscriber activates Pro Monthly, the Legacy $7.99 card is removed from their pricing view. Same for existing annual users who activate Pro Annual.
- **`replacementBehavior: "APPLY_IMMEDIATELY"`** is set for same-interval upgrades from an active paid plan. Shopify credits the remaining balance and switches the plan immediately.
- **Cross-interval upgrades** rely on Shopify's default behavior (no `replacementBehavior`).
- **Partner discount on Pro Annual** is applied server-side based on the user's partner referral status — no separate coupon needed.

**Edge cases:**
- Legacy monthly → Pro Annual (cross-interval): `replacementBehavior` omitted → Shopify default.
- Legacy annual → Pro Monthly (cross-interval): `replacementBehavior` omitted → Shopify default.
- Trial days: same 7-day trial logic; users who exhausted their trial get 0 days.
- Coupon entered for Annual or Standard: backend rejects, price unchanged.

---

## 4. End-to-End Flow

### 4a. Merchant Segment: New User / Existing Free

**Pricing page shows 3 cards:**
```
[ Free $0/mo (Current) ]  [ Plus Plan $9.99/mo ]  [ Pro Annual $95.90/yr ]
```

> The $9.99 plan is displayed as **"Plus Plan"** (frontend label only; internal DB ID remains `bucks_premium_standard`).
> The Pro $19.99 monthly plan is **not shown** to free/new users.

**Merchant flow:**
1. Opens Pricing page → sees 3 cards
2. Clicks "Start Trial" on Plus Plan → redirected to Shopify billing confirmation
3. Approves → `checkSubscriptionStatus` writes `bucks_plan: "bucks_premium_standard"` to DB
4. Redirected back to pricing page with success message

---

### 4b. Merchant Segment: Existing $7.99 Monthly

**Pricing page shows 4 cards:**
```
[ Free $0/mo ]  [ Legacy $7.99/mo (Current) ]  [ Pro $19.99/mo ]  [ Pro Annual $95.90/yr ]
```

> Standard $9.99 card is **not shown** to existing paid users.

**Same-interval upgrade (Monthly → Pro $19.99):**
1. Merchant opens Pro card → enters `BUCKS33` in the coupon input on the card → price updates instantly to $9.99 in the UI
2. Clicks upgrade → `initSubscription` detects: `currentPlan = "bucks_premium"` (monthly), `newPlan interval = monthly` → same interval
3. Sets `replacementBehavior: "APPLY_IMMEDIATELY"` in Shopify mutation
4. Shopify: credits remaining $7.99 balance, switches plan immediately
5. Merchant approves → DB updated with `bucks_plan: "bucks_premium_pro"`
6. **Legacy $7.99 card is removed** from pricing page for this merchant going forward

**Cross-interval upgrade (Monthly → Pro Annual $95.90):**
1. `replacementBehavior` is omitted → Shopify default
2. Old monthly removed when new annual activates
3. **Legacy $7.99 card is removed** from pricing page once new annual is active

---

### 4c. Merchant Segment: Existing Annual Subscriber

**Pricing page shows 4 cards:**
```
[ Free $0/mo ]  [ Pro $19.99/mo ]  [ Legacy Annual (Current) ]  [ Pro Annual $95.90/yr ]
```

> Standard $9.99 card is **not shown** to existing paid users.

**Same-interval upgrade (Annual → Pro Annual $95.90):**
1. Merchant clicks upgrade on Pro Annual card
2. `initSubscription` detects: same interval (annual → annual)
3. Sets `replacementBehavior: "APPLY_IMMEDIATELY"` → Shopify credits remaining annual balance and switches immediately
4. **Legacy Annual card is removed** from pricing page for this merchant going forward

**Cross-interval upgrade (Annual → Pro Monthly $19.99):**
1. Merchant opens Pro Monthly card → enters `BUCKS33` → price shows $9.99 in UI
2. `replacementBehavior` omitted (cross-interval) → Shopify default
3. Legacy Annual continues until replaced when new monthly activates

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
  │    - replacementBehavior = (isOnActivePaidPlan && isSameInterval) ? "APPLY_IMMEDIATELY" : undefined
  │
  ├─ Validate coupon (Pro Monthly only)
  │    - if newPlan == "bucks_premium_pro" && couponCode === "BUCKS33"
  │        → override price.amount = 9.99  (50.03% off $19.99)
  │    - if newPlan == "bucks_premium_pro_annual" OR "bucks_premium_standard"
  │        → ignore couponCode entirely
  │
  ├─ Apply partner discount (Pro Annual only)
  │    - if newPlan == "bucks_premium_pro_annual" && userRecord.isPartner
  │        → price.amount = 83.93  (34.98% of $239.88)
  │    - else
  │        → price.amount = 95.90  (39.98% of $239.88)
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
    replacementBehavior: $replacementBehavior  # passed only when APPLY_IMMEDIATELY
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
export const NEW_STANDARD_MONTHLY_PRICE = 9.99;           // Standard — fixed price for free/new users
export const NEW_PRO_MONTHLY_PRICE = 19.99;               // Pro Monthly — full price before coupon
export const NEW_PRO_MONTHLY_PRICE_WITH_COUPON = 9.99;    // Pro Monthly — after BUCKS33 (50.03% off)
export const NEW_PRO_ANNUAL_PRICE = 95.90;                // Pro Annual — default (39.98% of $239.88)
export const NEW_PRO_ANNUAL_PRICE_PARTNER = 83.93;        // Pro Annual — partner (34.98% of $239.88)
export const NEW_PRO_ANNUAL_FULL_PRICE = 239.88;          // Pro Annual — compare/strikethrough price
export const PRO_COUPON_CODE = "BUCKS33";                  // Pro Monthly only; 50.03% off
export const PRO_COUPON_DISCOUNT_PERCENT = 50.03;
```

Three new `planDetails` entries with IDs: `"standard"`, `"pro"`, `"pro-annual"`.

**`pages/pricing/index.jsx`**
- Build `visiblePlanIds` array per user segment:
  - Free/New: `["free", "standard", "pro-annual"]` — **3 cards**
  - Legacy Monthly (active): `["free", "legacy-monthly", "pro", "pro-annual"]` — **4 cards**
  - Legacy Monthly (upgraded to Pro Monthly): `["free", "pro", "pro-annual"]` — **3 cards** (legacy hidden)
  - Legacy Annual (active): `["free", "pro", "legacy-annual", "pro-annual"]` — **4 cards**
  - Legacy Annual (upgraded to Pro Annual): `["free", "pro", "pro-annual"]` — **3 cards** (legacy hidden)
- Legacy card visibility rule: hide legacy card when `bucks_plan` is one of the new Pro plan IDs
- Dynamic `InlineGrid` columns: `mdDown ? 1 : lgDown ? 2 : cardCount`

**`components/pricing/PricingCard.jsx`**
- **Plan display name**: render `plan.displayName` (e.g. `"Plus Plan"` for free/new users, `"Standard"` for existing paid users) — pass as prop from parent based on user segment
- **Coupon input field**: rendered only on the Pro Monthly card (`plan.couponCode` exists)
  - Input + "Apply" button inline on the card
  - On apply: validates code client-side → if valid, immediately updates displayed price to $9.99, shows `~~$19.99~~` struck-through
  - Coupon state stored in component; sent as `couponCode` in the `initSubscription` API call
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
| `"bucks standard plan initiated"` | Standard/Plus $9.99 selected (free/new users) |
| `"bucks pro monthly initiated"` | Pro $19.99 card clicked |
| `"bucks pro coupon applied"` | BUCKS33 applied on card (track: shop, finalPrice: 9.99) |
| `"bucks pro annual initiated"` | Pro Annual $95.90 or $83.93 card clicked |
| `"bucks plan activated"` | Shopify billing callback confirms any new plan (track: shop, plan, price) |

---

## 6. Open Questions

- **Coupon expiry:** Is `BUCKS33` always active or time-limited? If time-limited, store in env var.
- **Analytics teaser UX:** Blur overlay vs. locked rows vs. date-limited view? Needs design input.
- **Grid responsiveness (3–4 cards):** On tablet, single-row scroll or 2-col wrap? Confirm breakpoints.
- **Legacy plan CTA copy:** "Upgrade to Pro" vs. "Switch to Pro" vs. "Unlock Analytics"?
- **Partner detection:** How is `isPartner` flag determined server-side for Pro Annual discount? Via existing partner referral table?

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
| **Phase 2** | Backend: `createAppSubscription` (APPLY_IMMEDIATELY), `initSubscription` (coupon + partner discount) | `createAppSubscription.js`, `initSubscription.js` | 1 day |
| **Phase 3** | Frontend: Pricing page segment filtering, dynamic grid, legacy card hide logic, display name per segment, coupon input UI on card | `pages/pricing/index.jsx`, `PricingCard.jsx` | 1.5 days |
| **Phase 4** | Analytics gate: hasFullAnalytics check + teaser UI | `pages/analytics/index.jsx` | 0.5 day |
| **Phase 5** | QA: Test all 3 user segments, coupon apply flow, APPLY_IMMEDIATELY paths, legacy card hide | — | 1 day |

Each phase can be shipped independently. Phase 1 is a prerequisite for all others.
