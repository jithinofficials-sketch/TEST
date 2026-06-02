# New Pricing Plans — Bucks Currency Converter

**Document created:** June 2026  
**Status:** Planning / Pre-implementation

---

## 1. Overview

We are introducing a new **Pro tier** priced at **$14.99/month** (and **$179.88/year**) alongside the existing $7.99/month and discounted annual plans.

The new plans are the **going-forward standard paid tier** for new users. Existing users retain access to their current plans but are shown the new plans as upgrade options.

The key differentiator of new plans: **Full Analytics access** (existing plans get partial/teaser view).

---

## 2. Plan Catalogue

| Plan ID (DB) | Display Name | Price | Interval | Who sees it |
|---|---|---|---|---|
| `bucks_free` | Free | $0 | — | Everyone |
| `bucks_premium` | Plus Monthly *(legacy)* | $7.99/mo | Monthly | Existing $7.99 subscribers only |
| `bucks_premium_60/65/70` | Plus Yearly *(legacy)* | varies | Annual | Existing annual subscribers only |
| **`bucks_premium_pro`** | **Pro Monthly** | **$14.99/mo** | Monthly | Everyone |
| **`bucks_premium_pro_annual`** | **Pro Yearly** | **$179.88/yr** | Annual | Everyone |

### Coupon Discount (Pro Plans Only)

- **Code:** Fixed hardcoded code (e.g. `BUCKS33`) — displayed directly on the pricing card
- **Monthly discount:** $14.99 → **$9.99/month** (~33.36% off)
- **Yearly discount:** $179.88 → **$119.88/year** ($9.99/month × 12) — same percentage applied
- **Validation:** Backend validates the code and reduces `price.amount` before calling the Shopify subscription mutation; no native Shopify coupon involved
- **Scope:** Only applicable to new Pro plans; not applicable to legacy $7.99 plans

---

## 3. Plan Display by User Segment

The pricing grid adapts based on the current user's plan. Layout changes from 3-column to 4-column (or 2×2 on mobile) when 4 cards are shown.

### 3a. New Users & Existing Free Users

**Cards shown (3 cards):**

```
[ Free ] [ Pro Monthly $14.99 ] [ Pro Yearly $179.88 ]
```

- Free card: "Downgrade" / "Current plan"
- Pro Monthly: coupon code shown → $9.99 after code
- Pro Yearly: coupon code shown → $119.88 after code
- Trial: 7-day free trial applies for Pro Monthly (existing trial logic)

---

### 3b. Existing Monthly Plan Users ($7.99/month — `bucks_premium`)

**Cards shown (4 cards):**

```
[ Free ] [ Legacy Monthly $7.99 ] [ Pro Monthly $14.99 ] [ Pro Yearly $179.88 ]
```

- Legacy Monthly: shows "Current plan" badge; user cannot re-subscribe
- Pro Monthly ($14.99): uses `replacementBehavior: "STANDARD"` → remaining balance on $7.99 plan is credited; old plan is removed
- Pro Yearly ($179.88): uses `replacementBehavior: "STANDARD"` → remaining monthly balance credited toward yearly plan
- Coupon code visible on both new plan cards

#### Shopify Mutation — Upgrade from Monthly → Pro Monthly

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
    replacementBehavior: $replacementBehavior  # STANDARD
  ) {
    userErrors { field message }
    appSubscription { id }
    confirmationUrl
  }
}
```

Variables:
```json
{
  "replacementBehavior": "STANDARD"
}
```

---

### 3c. Existing Annual Plan Users (`bucks_premium_60/65/70`)

**Cards shown (4 cards):**

```
[ Free ] [ Pro Monthly $14.99 ] [ Legacy Yearly (their price) ] [ Pro Yearly $179.88 ]
```

- Legacy Yearly: shows "Current plan" badge
- Pro Monthly ($14.99): uses `replacementBehavior: "STANDARD"` → remaining annual balance credited
- Pro Yearly ($179.88): uses `replacementBehavior: "STANDARD"` → remaining annual balance credited; old annual plan removed
- Coupon code visible on both new plan cards

---

## 4. New Plan Database Identifiers

### New `bucks_plan` values

| DB Value | Description |
|---|---|
| `bucks_premium_pro` | New Pro Monthly ($14.99/month) |
| `bucks_premium_pro_annual` | New Pro Yearly ($179.88/year) |

### `PREMIUM_PLANS` constant update

```js
// utils/constants.js
export const PREMIUM_PLANS = {
  MONTHLY: "bucks_premium",             // legacy $7.99
  ANNUAL_60: "bucks_premium_60",        // legacy annual 60% discount
  ANNUAL_65: "bucks_premium_65",        // legacy annual 65% (partner)
  ANNUAL_70: "bucks_premium_70",        // legacy annual 70% (BFCM)
  PRO_MONTHLY: "bucks_premium_pro",     // NEW $14.99/month
  PRO_ANNUAL: "bucks_premium_pro_annual", // NEW $179.88/year
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
  "bucks_premium_pro",          // NEW
  "bucks_premium_pro_annual",   // NEW
];
```

---

## 5. Pricing Constants

```js
// utils/common/constants/constants.js

export const NEW_PRO_MONTHLY_PRICE = 14.99;
export const NEW_PRO_ANNUAL_PRICE = 179.88; // 14.99 * 12

export const PRO_COUPON_CODE = "BUCKS33"; // displayed on card
export const PRO_COUPON_DISCOUNT_PERCENT = 33.36; // ~33.36%
// Monthly with coupon: $9.99  | Annual with coupon: $119.88

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
1. Receives payload with `{ couponCode, price: { amount }, ... }`
2. Validates `couponCode === PRO_COUPON_CODE`
3. If valid: overrides `price.amount` with discounted amount before calling `createAppSubscription`
4. Also passes `replacementBehavior: "STANDARD"` when user is on an active premium plan

### Example API Payload (Pro Monthly with coupon)
```json
{
  "name": "Pro Monthly",
  "plan_name": "bucks_premium_pro",
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

Used only when user **currently has an active paid subscription** and is switching to a new plan.

```js
// In createAppSubscription.js / initSubscription.js
const isUpgradingFromPaidPlan = Boolean(userRecord?.subscription_id) && isPremiumPlan(userRecord?.bucks_plan);

const replacementBehavior = isUpgradingFromPaidPlan ? "STANDARD" : undefined;
// "STANDARD" = remaining balance credited, old subscription removed immediately
```

| From → To | replacementBehavior |
|---|---|
| Free → Pro Monthly | `undefined` (no active paid sub) |
| Free → Pro Yearly | `undefined` |
| Legacy Monthly ($7.99) → Pro Monthly ($14.99) | `"STANDARD"` |
| Legacy Monthly ($7.99) → Pro Yearly ($179.88) | `"STANDARD"` |
| Legacy Annual → Pro Monthly | `"STANDARD"` |
| Legacy Annual → Pro Yearly | `"STANDARD"` |

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
| `bucks_free` | Partial / teaser view (locked rows, blur overlay, or limited date range) |
| `bucks_premium` (legacy $7.99) | Partial / teaser view |
| `bucks_premium_60/65/70` (legacy annual) | Partial / teaser view |
| **`bucks_premium_pro`** | **Full analytics** |
| **`bucks_premium_pro_annual`** | **Full analytics** |

The analytics page (`pages/analytics/index.jsx`) should check:
```js
const hasFullAnalytics = ["bucks_premium_pro", "bucks_premium_pro_annual"].includes(userData?.bucks_plan);
```

If `!hasFullAnalytics`: show a partial/teaser view with a prompt to upgrade to Pro plans.

---

## 10. `planDetails` Array — New Entries

Two new entries added to `planDetails` in `utils/common/constants/constants.js`:

```js
{
  id: "pro-v2",
  plan_name: "bucks_premium_pro",
  plan_type: "monthly",
  action: "Upgrade",
  name: "Pro Monthly",
  description: "Full analytics + all premium features at our new standard price",
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
    comparePrice: 14.99,
  },
  interval: MONTHLY_INTERVAL,
  frequency: "month",
  trialDays: DEFAULT_TRIAL_DAYS,
  buttonText: "Start your 7-day free trial",
  billingNote: "Billed monthly • No commitment",
  footerText: "Then $14.99/month · Use code BUCKS33 for $9.99/month",
  couponCode: "BUCKS33",
  couponPrice: 9.99,
},
{
  id: "pro-v2-annual",
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
    comparePrice: 179.88,
  },
  interval: ANNUAL_INTERVAL,
  frequency: "year",
  trialDays: DEFAULT_TRIAL_DAYS,
  buttonText: "Start your 7-day free trial",
  billingNote: "Billed annually",
  footerText: "Use code BUCKS33 for $119.88/year ($9.99/month)",
  couponCode: "BUCKS33",
  couponPrice: 119.88,
  hasBlueBorder: true,
},
```

---

## 11. Pricing Page Grid Layout

Currently `InlineGrid columns={mdDown ? 1 : 3}`.

With 4 cards (for monthly/annual legacy users), this needs to change:

```jsx
// Determine column count dynamically
const cardCount = visiblePlans.length; // 3 or 4
const columns = mdDown ? 1 : (cardCount === 4 ? 4 : 3);

<InlineGrid gap="400" columns={columns} alignItems="stretch">
```

On tablet/medium screens, 4 columns may be cramped — consider `lgDown ? 2 : 4` for 4-card layouts.

---

## 12. `checkSubscriptionStatus.js` — Plan Type Update

The query param `bucks_plan` already flows through the return URL. New values `bucks_premium_pro` and `bucks_premium_pro_annual` will be passed automatically via the existing mechanism in `createAppSubscription.js`:

```
returnUrl: `...&bucks_plan=bucks_premium_pro&plan_type=monthly`
```

No change needed to `checkSubscriptionStatus.js` — it already reads these from query params and saves to DB.

---

## 13. `getPlanType` Helper Update

```js
// utils/common/common-methods/pricingConfig.js
export function getPlanType(user) {
  if (user?.bucks_plan === "bucks_premium") return user?.plan_type || "monthly";
  if (user?.bucks_plan === "bucks_premium_pro") return "monthly";
  if (user?.bucks_plan === "bucks_premium_pro_annual") return "annual";
  if (PREMIUM_PLAN_IDS.slice(1).includes(user?.bucks_plan)) return "annual";
  return user?.plan_type || "";
}
```

---

## 14. Implementation Checklist

### Constants & Config
- [ ] Add `NEW_PRO_MONTHLY_PRICE`, `NEW_PRO_ANNUAL_PRICE`, `PRO_COUPON_CODE`, `PRO_COUPON_DISCOUNT_PERCENT` to `utils/common/constants/constants.js`
- [ ] Add `PRO_MONTHLY` and `PRO_ANNUAL` to `PREMIUM_PLANS` in `utils/constants.js`
- [ ] Add `bucks_premium_pro` and `bucks_premium_pro_annual` to `PREMIUM_PLAN_IDS` in `trialHelpers.js`
- [ ] Add two new entries to `planDetails` array

### Backend
- [ ] Update `createAppSubscription.js` to accept and pass `replacementBehavior` param
- [ ] Update `initSubscription.js` to:
  - Detect when user is upgrading from paid plan → set `replacementBehavior: "STANDARD"`
  - Validate coupon code → apply discounted price
  - Set correct `bucks_plan` for new Pro plans
- [ ] Update `getPlanType` in `pricingConfig.js`
- [ ] Ensure `checkSubscriptionStatus.js` handles new plan IDs (no code change needed)

### Frontend
- [ ] Update `planDetails` in constants with new plan objects
- [ ] Update pricing page to filter/show correct cards per user segment
- [ ] Update `InlineGrid` columns to handle 3 or 4 cards dynamically
- [ ] Add coupon code UI to Pro plan cards (input field or inline display with copy)
- [ ] Update `PricingCard.jsx` to show coupon info and discounted price

### Analytics Gate
- [ ] Update `pages/analytics/index.jsx` to check for `bucks_premium_pro` / `bucks_premium_pro_annual`
- [ ] Add partial/teaser view for non-Pro plans with upgrade prompt

### PostHog Events
- [ ] `"bucks pro plan initiated"` — track when Pro Monthly or Pro Yearly selected
- [ ] `"bucks pro coupon applied"` — track coupon usage with code and final amount
- [ ] `"bucks pro plan selected"` — track after Shopify confirms subscription

---

## 15. Open Questions / Decisions Pending

- [ ] **Exact coupon code string** — placeholder used `BUCKS33`; confirm final code
- [ ] **Analytics teaser UX** — blur overlay vs locked rows vs date-limited view? Design needed
- [ ] **4-card grid responsiveness** — confirm breakpoints for tablet view (2×2 vs 4 columns)
- [ ] **Legacy plan users' upgrade CTA copy** — e.g. "Upgrade to Pro" vs "Switch to Pro"
- [ ] **Prisma schema** — no new DB columns needed (`bucks_plan` and `plan_type` already exist); confirm with team
- [ ] **Whether legacy annual users can downgrade to legacy monthly** — currently not shown to them; should it be?

---

*This document should be updated as decisions are finalized. Implementation should follow the checklist in Section 14 in order.*
