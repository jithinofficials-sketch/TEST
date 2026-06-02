# New Pricing Plans — Bucks Currency Converter

**Document created:** June 2026  
**Last updated:** June 2026 (rev 2)  
**Status:** Planning / Pre-implementation

---

## 1. Overview

We are introducing **two new paid tiers** alongside the existing $7.99/month and legacy annual plans:

- **Standard Monthly — $9.99/month** (`bucks_premium_standard`): All current Plus features, no analytics. Replaces the legacy $7.99 plan as the base upgrade path.
- **Pro Monthly — $14.99/month** (`bucks_premium_pro`): All Standard features **+ Full Analytics**. Coupon reduces to $9.99/mo.
- **Pro Annual — $179.88/year** (`bucks_premium_pro_annual`): Same as Pro Monthly billed annually. Coupon reduces to $119.88/yr.

Legacy plans ($7.99/mo, discounted annual) remain accessible only to their current subscribers. New users and free users do NOT see legacy plan cards.

---

## 2. Plan Catalogue

| Plan ID (DB) | Display Name | Price | Interval | Analytics | Status |
|---|---|---|---|---|---|
| `bucks_free` | Free | $0 | — | None | Active |
| `bucks_premium` | Plus Monthly *(legacy)* | $7.99/mo | Monthly | None | Legacy — existing subscribers only |
| `bucks_premium_60/65/70` | Plus Yearly *(legacy)* | varies | Annual | None | Legacy — existing subscribers only |
| **`bucks_premium_standard`** | **Standard Monthly** | **$9.99/mo** | Monthly | None | **NEW** |
| **`bucks_premium_pro`** | **Pro Monthly** | **$14.99/mo** | Monthly | **Full** | **NEW** |
| **`bucks_premium_pro_annual`** | **Pro Yearly** | **$179.88/yr** | Annual | **Full** | **NEW** |

### Coupon Discount (Pro Plans Only — NOT Standard)

- **Code:** Fixed hardcoded code (e.g. `BUCKS33`) — displayed directly on the Pro plan pricing cards
- **Pro Monthly discount:** $14.99 → **$9.99/month** (~33.36% off)
- **Pro Annual discount:** $179.88 → **$119.88/year** ($9.99 × 12) — same percentage applied
- **Validation:** Backend validates the code and reduces `price.amount` before the Shopify subscription mutation
- **Standard Monthly ($9.99):** No coupon — fixed price always $9.99

---

## 3. Plan Display by User Segment

The pricing grid adapts based on the current user's plan. Grid layout must handle **4 or 5 cards** for existing paid users.

### 3a. New Users & Existing Free Users — **4 cards**

```
[ Free ] [ Standard $9.99/mo ] [ Pro $14.99/mo ] [ Pro Annual $179.88/yr ]
```

- Free: "Current plan" badge
- Standard $9.99: 7-day trial; no coupon
- Pro $14.99: 7-day trial; coupon code shown → $9.99 after code
- Pro Annual $179.88: 7-day trial; coupon code shown → $119.88 after code

---

### 3b. Existing Monthly Users ($7.99/mo — `bucks_premium`) — **5 cards**

```
[ Free ] [ Legacy $7.99/mo ] [ Standard $9.99/mo ] [ Pro $14.99/mo ] [ Pro Annual $179.88/yr ]
```

- **Legacy $7.99:** "Current plan" badge — cannot re-subscribe
- **Standard $9.99:** `replacementBehavior: "STANDARD"` → remaining $7.99 balance credited; old plan removed immediately
- **Pro $14.99:** `replacementBehavior: "STANDARD"` → remaining $7.99 balance credited; old plan removed immediately
- **Pro Annual $179.88:** No `replacementBehavior` (cross-interval) — Shopify default handles transition; old monthly removed when new annual activates
- Coupon shown only on Pro $14.99 and Pro Annual cards

#### Shopify Mutation — Same-interval upgrade (Monthly → Standard or Pro Monthly)

```graphql
mutation AppSubscriptionCreate(
  $name: String!
  $lineItems: [AppSubscriptionLineItemInput!]!
  $returnUrl: URL!
  $trialDays: Int!
  $replacementBehavior: AppSubscriptionReplacementBehavior
) {
  appSubscriptionCreate(
    name: $name
    returnUrl: $returnUrl
    lineItems: $lineItems
    test: false
    trialDays: $trialDays
    replacementBehavior: $replacementBehavior
  ) {
    userErrors { field message }
    appSubscription { id }
    confirmationUrl
  }
}
```

Variables (same-interval only):
```json
{ "replacementBehavior": "STANDARD" }
```

---

### 3c. Existing Annual Users (`bucks_premium_60/65/70`) — **5 cards**

```
[ Free ] [ Standard $9.99/mo ] [ Pro $14.99/mo ] [ Legacy Yearly ] [ Pro Annual $179.88/yr ]
```

- **Standard $9.99:** No `replacementBehavior` (cross-interval annual→monthly) — Shopify default
- **Pro $14.99:** No `replacementBehavior` (cross-interval annual→monthly) — Shopify default
- **Legacy Yearly:** "Current plan" badge — cannot re-subscribe
- **Pro Annual $179.88:** `replacementBehavior: "STANDARD"` → remaining annual balance credited; old annual plan removed immediately
- Coupon shown only on Pro $14.99 and Pro Annual cards

---

## 4. New Plan Database Identifiers

### New `bucks_plan` values

| DB Value | Display Name | Price |
|---|---|---|
| `bucks_premium_standard` | Standard Monthly | $9.99/month |
| `bucks_premium_pro` | Pro Monthly | $14.99/month |
| `bucks_premium_pro_annual` | Pro Yearly | $179.88/year |

> No new database columns needed — `bucks_plan` and `plan_type` fields already exist in the `users` table.

### `PREMIUM_PLANS` constant update

```js
// utils/constants.js
export const PREMIUM_PLANS = {
  MONTHLY: "bucks_premium",                   // legacy $7.99
  ANNUAL_60: "bucks_premium_60",              // legacy annual 60% discount
  ANNUAL_65: "bucks_premium_65",              // legacy annual 65% (partner)
  ANNUAL_70: "bucks_premium_70",              // legacy annual 70% (BFCM)
  STANDARD_MONTHLY: "bucks_premium_standard", // NEW $9.99/month
  PRO_MONTHLY: "bucks_premium_pro",           // NEW $14.99/month
  PRO_ANNUAL: "bucks_premium_pro_annual",     // NEW $179.88/year
};
```

### `PREMIUM_PLAN_IDS` update

```js
// utils/common/common-methods/trialHelpers.js
export const PREMIUM_PLAN_IDS = [
  "bucks_premium",
  "bucks_premium_60",
  "bucks_premium_65",
  "bucks_premium_70",
  "bucks_premium_standard",     // NEW
  "bucks_premium_pro",          // NEW
  "bucks_premium_pro_annual",   // NEW
];
```

---

## 5. Pricing Constants

```js
// utils/common/constants/constants.js

export const NEW_STANDARD_MONTHLY_PRICE = 9.99;   // Standard plan (no coupon)
export const NEW_PRO_MONTHLY_PRICE = 14.99;        // Pro plan full price
export const NEW_PRO_ANNUAL_PRICE = 179.88;        // 14.99 * 12

export const PRO_COUPON_CODE = "BUCKS33";           // displayed on Pro plan cards only
export const PRO_COUPON_DISCOUNT_PERCENT = 33.36;  // ~33.36%
// Pro Monthly with coupon: $9.99 | Pro Annual with coupon: $119.88

export const NEW_PRO_MONTHLY_PRICE_WITH_COUPON = 9.99;
export const NEW_PRO_ANNUAL_PRICE_WITH_COUPON = 119.88; // 9.99 * 12
```

---

## 6. Coupon Code Flow (Frontend → Backend)

### Frontend
1. Pricing card for Pro plans shows a coupon input field **or** shows the code inline with a "Copy" button
2. If user enters/applies the code before clicking upgrade, the discounted amount is sent in the API payload
3. The pricing card shows both the original price and the discounted price with the code

### Backend (`pages/api/v1/billing/initSubscription.js`)
1. Receives payload with `{ couponCode, price: { amount }, plan_type, bucks_plan, ... }`
2. If `bucks_plan` is `bucks_premium_pro` or `bucks_premium_pro_annual`: validates `couponCode === PRO_COUPON_CODE`
3. If valid: overrides `price.amount` with discounted amount before calling `createAppSubscription`
4. Determines `replacementBehavior` based on upgrade path (see Section 7)

### Example API Payloads

**Standard Monthly (no coupon):**
```json
{
  "name": "Standard Monthly",
  "plan_type": "monthly",
  "price": { "amount": 9.99, "currencyCode": "USD" },
  "interval": "EVERY_30_DAYS",
  "trialDays": 7,
  "bucks_plan": "bucks_premium_standard"
}
```

**Pro Monthly with coupon:**
```json
{
  "name": "Pro Monthly",
  "plan_type": "monthly",
  "price": { "amount": 14.99, "currencyCode": "USD" },
  "interval": "EVERY_30_DAYS",
  "trialDays": 7,
  "couponCode": "BUCKS33",
  "bucks_plan": "bucks_premium_pro"
}
```
Backend resolves final `price.amount = 9.99` before mutation.

---

## 7. `replacementBehavior` Logic

`replacementBehavior: "STANDARD"` is used **only for same-interval upgrades** from an active paid plan:
- Monthly plan → any new Monthly plan (Standard $9.99 or Pro $14.99)
- Annual plan → new Pro Annual ($179.88)

Cross-interval changes (monthly→annual, annual→monthly) use Shopify's default behavior — same as the current codebase handles monthly→annual today.

```js
// In initSubscription.js
const isOnActivePaidPlan = Boolean(userRecord?.subscription_id) && isPremiumPlan(userRecord?.bucks_plan);
const currentInterval = getPlanType(userRecord); // "monthly" or "annual"
const newInterval = subscriptionDetailsFromBody?.plan_type; // "monthly" or "annual"
const isSameInterval = currentInterval === newInterval;

const replacementBehavior = (isOnActivePaidPlan && isSameInterval) ? "STANDARD" : undefined;
```

| From Plan | From Interval | To Plan | To Interval | replacementBehavior |
|---|---|---|---|---|
| Free | — | Standard $9.99 | Monthly | `undefined` |
| Free | — | Pro $14.99 | Monthly | `undefined` |
| Free | — | Pro Annual $179.88 | Annual | `undefined` |
| Legacy $7.99 | Monthly | Standard $9.99 | Monthly | **`"STANDARD"`** |
| Legacy $7.99 | Monthly | Pro $14.99 | Monthly | **`"STANDARD"`** |
| Legacy $7.99 | Monthly | Pro Annual $179.88 | Annual | `undefined` (cross-interval) |
| Legacy Annual | Annual | Standard $9.99 | Monthly | `undefined` (cross-interval) |
| Legacy Annual | Annual | Pro $14.99 | Monthly | `undefined` (cross-interval) |
| Legacy Annual | Annual | Pro Annual $179.88 | Annual | **`"STANDARD"`** |

---

## 8. Trial Days Logic

Same existing logic — no changes:
- New users (never subscribed): 7-day free trial
- Users who used trial before: no trial (0 days)
- Users with remaining trial from a previous premium subscription: carry over

Applies to both Pro Monthly and Pro Yearly.

---

## 9. Analytics Feature Gate

| Plan | Analytics Access |
|---|---|
| `bucks_free` | None / teaser with upgrade prompt |
| `bucks_premium` (legacy $7.99 monthly) | None / teaser with upgrade prompt |
| `bucks_premium_60/65/70` (legacy annual) | None / teaser with upgrade prompt |
| `bucks_premium_standard` ($9.99 monthly) | None / teaser with upgrade prompt |
| **`bucks_premium_pro`** ($14.99 monthly) | **Full analytics** |
| **`bucks_premium_pro_annual`** ($179.88 annual) | **Full analytics** |

The analytics page (`pages/analytics/index.jsx`) should check:
```js
const PRO_ANALYTICS_PLANS = ["bucks_premium_pro", "bucks_premium_pro_annual"];
const hasFullAnalytics = PRO_ANALYTICS_PLANS.includes(userData?.bucks_plan);
```

If `!hasFullAnalytics`: show a partial/teaser view with an upgrade prompt pointing to the Pro plans. Design of teaser (blur overlay / locked rows / date-limited) is TBD.

---

## 10. `planDetails` Array — New Entries

Three new entries added to `planDetails` in `utils/common/constants/constants.js`:

```js
// Entry 1 — Standard Monthly ($9.99, no coupon, no analytics)
{
  id: "standard",
  plan_name: "bucks_premium_standard",
  plan_type: "monthly",
  action: "Upgrade",
  name: "Standard Monthly",
  description: "All premium features at an affordable price",
  features: [
    "All free features",
    "Unlimited conversions",
    "Unlimited page views",
    "Set manual conversion rates",
    "Customize currency display",
    "Priority support",
    "Third party apps support",
    "Custom positioning",
  ],
  unlimitedFeatures: ["Unlimited conversions", "Unlimited page views"],
  price: {
    amount: 9.99,
    currencyCode: "USD",
  },
  interval: MONTHLY_INTERVAL,
  frequency: "month",
  trialDays: DEFAULT_TRIAL_DAYS,
  buttonText: `Start your ${DEFAULT_TRIAL_DAYS}-day free trial`,
  billingNote: "Billed monthly • No commitment",
  footerText: "Then $9.99/month · Cancel anytime",
},

// Entry 2 — Pro Monthly ($14.99, coupon → $9.99, full analytics)
{
  id: "pro",
  plan_name: "bucks_premium_pro",
  plan_type: "monthly",
  action: "Upgrade",
  name: "Pro Monthly",
  description: "Full analytics + all premium features",
  features: [
    "All free features",
    "Full Analytics dashboard",
    "Unlimited conversions",
    "Unlimited page views",
    "Set manual conversion rates",
    "Customize currency display",
    "Priority support",
    "Third party apps support",
    "Custom positioning",
  ],
  unlimitedFeatures: ["Unlimited conversions", "Unlimited page views"],
  price: {
    amount: 14.99,
    currencyCode: "USD",
  },
  interval: MONTHLY_INTERVAL,
  frequency: "month",
  trialDays: DEFAULT_TRIAL_DAYS,
  buttonText: `Start your ${DEFAULT_TRIAL_DAYS}-day free trial`,
  billingNote: "Billed monthly • No commitment",
  footerText: "Then $14.99/month · Use code BUCKS33 for $9.99/month",
  couponCode: "BUCKS33",
  couponPrice: 9.99,
},

// Entry 3 — Pro Annual ($179.88, coupon → $119.88, full analytics)
{
  id: "pro-annual",
  plan_name: "bucks_premium_pro_annual",
  plan_type: "annual",
  action: "Upgrade",
  name: "Pro Yearly",
  badge: "BEST VALUE",
  description: "Full analytics + all premium features, billed yearly",
  features: [
    "All free features",
    "Full Analytics dashboard",
    "Unlimited conversions",
    "Unlimited currencies",
    "Unlimited page views",
    "Set manual conversion rates",
    "Customize currency display",
    "Priority support (24/7)",
    "Third-party apps support",
    "Custom positioning",
    "Price increase protection (locked in for 12 months)",
  ],
  unlimitedFeatures: ["Unlimited conversions", "Unlimited currencies", "Unlimited page views"],
  price: {
    amount: 179.88,
    currencyCode: "USD",
  },
  interval: ANNUAL_INTERVAL,
  frequency: "year",
  trialDays: DEFAULT_TRIAL_DAYS,
  buttonText: `Start your ${DEFAULT_TRIAL_DAYS}-day free trial`,
  billingNote: "Billed annually",
  footerText: "Use code BUCKS33 for $119.88/year ($9.99/month)",
  couponCode: "BUCKS33",
  couponPrice: 119.88,
  hasBlueBorder: true,
},
```

> **Note on ordering in `planDetails`:** Keep legacy plans in the array but filter them per-segment at render time (see Section 11).

---

## 11. Pricing Page Grid Layout & Plan Filtering

### Plan filtering per user segment

```js
// pages/pricing/index.jsx
const isLegacyMonthly = currentPlan === "bucks_premium" && currentPlanType === "monthly";
const isLegacyAnnual = ["bucks_premium_60", "bucks_premium_65", "bucks_premium_70"].includes(currentPlan);

// Build the ordered list of plan IDs to display
let visiblePlanIds;
if (isLegacyMonthly) {
  // 5 cards: Free, Legacy $7.99, Standard $9.99, Pro $14.99, Pro Annual $179.88
  visiblePlanIds = ["free", "legacy-monthly", "standard", "pro", "pro-annual"];
} else if (isLegacyAnnual) {
  // 5 cards: Free, Standard $9.99, Pro $14.99, Legacy Yearly, Pro Annual $179.88
  visiblePlanIds = ["free", "standard", "pro", "legacy-annual", "pro-annual"];
} else {
  // 4 cards: Free, Standard $9.99, Pro $14.99, Pro Annual $179.88
  visiblePlanIds = ["free", "standard", "pro", "pro-annual"];
}

const visiblePlans = allPlanDetails.filter(p => visiblePlanIds.includes(p.id));
// Maintain order from visiblePlanIds
```

### Dynamic grid columns

Currently `InlineGrid columns={mdDown ? 1 : 3}`. With 4–5 cards this must be dynamic:

```jsx
const cardCount = visiblePlans.length; // 4 or 5
const columns = mdDown ? 1 : lgDown ? 2 : cardCount;

<InlineGrid gap="400" columns={columns} alignItems="stretch">
```

For 5 cards on a large screen: 5-column layout. On medium screens (`lgDown`): 2–3 columns with wrapping. On mobile: single column.

---

## 12. `checkSubscriptionStatus.js` — No Changes Needed

The query param `bucks_plan` already flows through the return URL via `createAppSubscription.js`:

```
returnUrl: `...&bucks_plan=bucks_premium_standard&plan_type=monthly`
returnUrl: `...&bucks_plan=bucks_premium_pro&plan_type=monthly`
returnUrl: `...&bucks_plan=bucks_premium_pro_annual&plan_type=annual`
```

`checkSubscriptionStatus.js` already reads `bucks_plan` and `plan_type` from query params and writes them to DB — no code change required.

---

## 13. `getPlanType` Helper Update

```js
// utils/common/common-methods/pricingConfig.js
export function getPlanType(user) {
  if (user?.bucks_plan === "bucks_premium") return user?.plan_type || "monthly";
  if (user?.bucks_plan === "bucks_premium_standard") return "monthly";  // NEW
  if (user?.bucks_plan === "bucks_premium_pro") return "monthly";        // NEW
  if (user?.bucks_plan === "bucks_premium_pro_annual") return "annual";  // NEW
  // Legacy annual plans
  if (["bucks_premium_60", "bucks_premium_65", "bucks_premium_70"].includes(user?.bucks_plan)) return "annual";
  return user?.plan_type || "";
}
```

---

## 14. Implementation Checklist

### Constants & Config
- [ ] Add `NEW_STANDARD_MONTHLY_PRICE`, `NEW_PRO_MONTHLY_PRICE`, `NEW_PRO_ANNUAL_PRICE`, `PRO_COUPON_CODE`, `PRO_COUPON_DISCOUNT_PERCENT` to `utils/common/constants/constants.js`
- [ ] Add `STANDARD_MONTHLY`, `PRO_MONTHLY`, `PRO_ANNUAL` to `PREMIUM_PLANS` in `utils/constants.js`
- [ ] Add `bucks_premium_standard`, `bucks_premium_pro`, `bucks_premium_pro_annual` to `PREMIUM_PLAN_IDS` in `trialHelpers.js`
- [ ] Add 3 new entries to `planDetails` array (`standard`, `pro`, `pro-annual`)

### Backend
- [ ] Update `createAppSubscription.js` to accept and forward `replacementBehavior` param in the GraphQL mutation
- [ ] Update `initSubscription.js` to:
  - Detect same-interval upgrade from paid plan → set `replacementBehavior: "STANDARD"`
  - Validate coupon code (only for Pro plans) → apply discounted `price.amount`
  - Map `bucks_plan` correctly for all 3 new plan IDs
- [ ] Update `getPlanType` in `pricingConfig.js` for new plan IDs
- [ ] `checkSubscriptionStatus.js` — no changes needed

### Frontend — Pricing Page
- [ ] Build `visiblePlans` filter logic per user segment (Section 11)
- [ ] Update `InlineGrid` to use dynamic `cardCount`-based columns
- [ ] Add coupon code UI to Pro Monthly and Pro Annual cards only
- [ ] Update `PricingCard.jsx` to conditionally render coupon info and struck-through price
- [ ] Handle `isCurrentPlanCard` check for all new plan IDs in `PricingCard.jsx`

### Analytics Gate
- [ ] Define `PRO_ANALYTICS_PLANS` constant
- [ ] Update `pages/analytics/index.jsx` to check `hasFullAnalytics`
- [ ] Build teaser/upgrade-prompt UI for non-Pro plans (design TBD)

### PostHog Events
- [ ] `"bucks standard plan initiated"` — Standard $9.99 selected
- [ ] `"bucks pro plan initiated"` — Pro $14.99 or Pro Annual $179.88 selected
- [ ] `"bucks pro coupon applied"` — track code used + final amount
- [ ] `"bucks pro plan selected"` — confirmed by Shopify billing callback

---

## 15. Open Questions / Decisions Pending

- [ ] **Exact coupon code string** — placeholder used `BUCKS33`; confirm final code
- [ ] **Analytics teaser UX** — blur overlay vs locked rows vs date-limited view? Design needed
- [ ] **5-card grid responsiveness** — confirm breakpoints: on medium screens, 2×3 wrap or horizontal scroll?
- [ ] **Legacy plan CTA copy** — e.g. "Upgrade to Pro" vs "Switch to Pro" vs "Unlock Analytics"
- [ ] **Standard plan display name** — "Standard Monthly" vs "Plus Monthly" vs "Essential"?
- [ ] **No annual version of Standard ($9.99)?** — Confirmed: only monthly $9.99; no $9.99×12 plan
- [ ] **Coupon expiry** — Is the coupon always active or time-limited? If time-limited, need env var for expiry date

---

*This document should be updated as decisions are finalized. Implementation should follow the checklist in Section 14 in order.*
