# Secure Billing Flow Design

## Goal

Fix the pricing and billing security issues without removing any existing plan. The merchant can still choose legacy monthly, legacy annual, Plus, Pro monthly, Pro annual, and partner Pro annual plans. The browser can choose only the plan ID and optional coupon code. The server owns the price, interval, trial, discount, and entitlement data.

## Current flow

1. The merchant opens `pages/pricing/index.jsx`.
2. `getServerSideProps()` loads merchant data with `loadShopData(shop)` and fetches partner data from `userData.partner_referral`.
3. The pricing page imports `planDetails` from `utils/common/constants/constants.js`.
4. The page decides which cards to show from `currentPlan`, `currentPlanType`, and `isLegacyUser`.
5. `computePlanProps()` calculates display values such as annual discount, compare price, badge, savings text, and trial text.
6. `PricingCard` receives the full frontend plan object through the `plan` prop.
7. When the merchant clicks upgrade, `PricingCard` calls `upgradePlanHandler({ ...plan, couponCode })`.
8. `upgradePlanHandler()` calculates trial days in the browser and sends `JSON.stringify({ ...plan, trialDays })` to `/api/v1/billing/initSubscription`.
9. `initSubscription.js` authenticates the merchant with `fetchSession()`, fetches user and partner data, and reads `req.body` as `subscriptionDetailsFromBody`.
10. New plans get a server-side amount map, but the code spreads the request body back into `finalDetails` and keeps client-controlled `interval`, `plan_type`, `name`, and `trialDays`.
11. Legacy plans keep the request body. Legacy monthly sends request price directly to Shopify. Legacy annual calculates discount from request `price.comparePrice`.
12. `createAppSubscription.js` builds the Shopify `appSubscriptionCreate` mutation from `subscriptionDetailsFromBody` and puts `shop`, `trialDays`, `plan_type`, and `bucks_plan` into the return URL.
13. After Shopify approval, Shopify redirects to `pages/api/v1/billing/checkSubscriptionStatus.js`.
14. `checkSubscriptionStatus.js` reads `shop`, `charge_id`, `trialDays`, `plan_type`, and `bucks_plan` from query params, checks that the Shopify charge is active, and writes those query values to `users`.
15. The callback syncs metafields with the new `bucks_plan`, `plan_type`, and `subscription_id`.

## Problems by issue

### #279 Annual subscription price is calculated from attacker-controlled base amount

The vulnerable path is `pages/api/v1/billing/initSubscription.js` legacy annual flow.

```js
const baseAmount = discountedSubscriptionDetails.price.comparePrice;

const { finalAmount, discountPercent } =
  calculateDiscountPricing(baseAmount, userRecord, partnerData);
```

`discountedSubscriptionDetails` comes from `req.body`. A merchant can replace `price.comparePrice` before the request reaches the API. The server then calculates the annual discount from the fake compare price.

The fix is to resolve the legacy annual plan from a backend catalog and pass the catalog compare price into discount calculation.

```js
const plan = getBillingPlan(req.body?.planId);
const discountPricing = plan.discountEligible
  ? calculateDiscountPricing(plan.comparePrice, userRecord, partnerData)
  : null;
```

### #270 New plan prices can be paired with attacker-controlled billing intervals

The vulnerable path is `pages/api/v1/billing/initSubscription.js` new-plan branch.

```js
const finalDetails = {
  ...subscriptionDetailsFromBody,
  price: planPrice,
  bucks_plan: planId,
};
```

The server replaces only `price`. It keeps `interval`, `plan_type`, `name`, and `trialDays` from the browser. `createAppSubscription.js` sends `subscriptionDetailsFromBody.interval` to Shopify.

The fix is to never spread the request body into subscription details. Build the object from catalog fields only.

```js
const trustedDetails = buildTrustedSubscriptionDetails({
  plan,
  userRecord,
  partnerData,
  couponOverride,
  discountPricing,
  trialDays,
});
```

### #269 Legacy subscription prices are calculated from client-controlled amounts

The vulnerable path is the legacy branch in `pages/api/v1/billing/initSubscription.js`.

```js
let discountedSubscriptionDetails = { ...subscriptionDetailsFromBody };
```

Monthly legacy billing does not canonicalize `price.amount`, `price.currencyCode`, or `interval`. Annual legacy billing also uses request `price.comparePrice`.

The fix is to route legacy and new plans through the same server catalog. Legacy plans stay available, but their billing terms come from the catalog.

### #268 Billing callback trusts attacker-controlled entitlement data

The vulnerable path is `pages/api/v1/billing/checkSubscriptionStatus.js`.

```js
const { charge_id, shop, trialDays = 0, plan_type = "monthly", bucks_plan = "bucks_premium" } = req.query;
```

The callback later writes those query values into the merchant record.

```js
await prisma.users.update({
  where: { myshopify_domain: shop },
  data: {
    bucks_plan,
    plan_type,
    trialEndDate: new Date(now.getTime() + trialDays * 24 * 60 * 60 * 1000),
    trialLeft: +trialDays,
    trialDays: +trialDays,
    subscription_id,
  },
});
```

The fix is to store a one-time billing state at initiation. The return URL carries only `state`. The callback consumes the state atomically and writes entitlements from the stored state, not from query params.

### #281 Client-controlled subscription price, interval, and trial duration reach Shopify

The vulnerable path is `utils/graphQl/createAppSubscription.js`.

```js
price: {
  amount: finalAmount,
  currencyCode: subscriptionDetailsFromBody.price.currencyCode,
},
interval: subscriptionDetailsFromBody.interval,
trialDays: trialDays,
```

This helper accepts a mixed object that can contain browser data. The fix is to make it a thin Shopify wrapper that receives trusted server-built details and a one-time billing state.

## Target flow

1. The pricing page still renders from `planDetails` for display.
2. When the merchant clicks upgrade, the browser sends only `{ planId, couponCode }`.
3. `initSubscription.js` authenticates with `fetchSession()`.
4. The endpoint looks up the plan ID in a backend catalog.
5. The endpoint fetches user and partner data from the database.
6. The endpoint validates partner-only access, coupon eligibility, discount eligibility, and trial eligibility.
7. The endpoint builds trusted subscription details from server-owned fields.
8. The endpoint creates a random billing state and passes it into the Shopify return URL.
9. Shopify receives price, currency, interval, name, and trial days from trusted details only.
10. After Shopify returns an app subscription ID and confirmation URL, the endpoint stores the pending billing state with the expected subscription ID and commercial terms.
11. The merchant approves billing in Shopify.
12. Shopify redirects to `/api/v1/billing/checkSubscriptionStatus?state=<token>`.
13. The callback loads and atomically consumes the pending state.
14. The callback loads the offline session for the stored shop, checks Shopify charge status, and confirms the subscription is active.
15. The callback writes `bucks_plan`, `plan_type`, accepted price, trial dates, and subscription ID from stored state.
16. The callback syncs metafields from stored state and redirects the merchant back to pricing.

## Backend plan catalog

Add `utils/billing/billingPlanCatalog.js`. It should include all current plans:

- `legacy-monthly`
- `legacy-annual`
- `plus`
- `pro-monthly`
- `pro-annual`
- `pro-annual-partner`

Each catalog entry owns:

- `id`
- `name`
- `bucksPlan`
- `planType`
- `amount`
- `comparePrice`
- `currencyCode`
- `interval`
- `discountEligible`
- `couponEligible`
- `partnerOnly`

The frontend `planDetails` can keep its current UI fields, but those fields are display-only.

## Trial policy

Trial days must be resolved server side. The browser must not send trial days.

Rules:

- Existing premium merchant with an active `trialEndDate` keeps the remaining days.
- Free merchant with `trialLeft > 0` keeps that preserved trial balance.
- Merchant with no `subscription_id` receives partner trial days or `DEFAULT_TRIAL_DAYS`.
- Merchant that already subscribed and has no remaining trial gets `0`.
- Trial days must be an integer from `0` to `90`.

## Billing state

Add a Prisma model for pending billing state.

```prisma
model billing_states {
  id             String    @map("_id") @id @default(auto()) @db.ObjectId
  state          String    @unique
  shop           String
  subscriptionId String?
  planId         String
  bucksPlan      String
  planType       String
  name           String
  amount         Float
  currencyCode   String
  interval       String
  trialDays      Int
  status         String    @default("pending")
  expiresAt      DateTime
  consumedAt     DateTime?
  createdAt      DateTime  @default(now())

  @@index([shop, status])
  @@index([expiresAt])
}
```

The state expires after 30 minutes. The callback consumes it with `updateMany` using `state`, `status: "pending"`, `consumedAt: null`, and `expiresAt > now`. If no row is updated, the callback fails.

## Shopify verification

The existing callback uses the REST recurring application charge endpoint. Keep this for the minimal fix because the current flow already depends on it.

The callback must still verify:

- Shopify response exists.
- Charge status is `active`.
- The active charge belongs to the stored shop session.
- The local entitlement update uses stored state values, not query values.

If Shopify returns an ID that differs from the stored subscription ID format, the callback should still store the subscription ID from the initiation response. The old callback already converts REST charge IDs into GraphQL IDs, so this fix should avoid making the ID format stricter than the current Shopify return behavior until we verify all callback shapes in production.

## Out of scope

- Removing legacy plans.
- Redesigning pricing cards.
- Changing public pricing copy.
- Implementing the `APP_SUBSCRIPTIONS_UPDATE` webhook.
- Migrating all billing verification from REST to Admin GraphQL.
- Changing cancellation behavior except where it depends on the corrected `subscription_id`.

## Test coverage

Add regression tests for:

- Unknown plan ID is rejected.
- Legacy monthly ignores submitted price, currency, interval, and trial days.
- Legacy annual ignores submitted compare price.
- New monthly cannot be created with annual interval.
- Partner-only annual plan rejects non-partner shops.
- Coupon applies only to Pro monthly.
- Trial days are computed from DB state.
- Callback ignores query `bucks_plan`, `plan_type`, and `trialDays`.
- Callback cannot consume the same state twice.
- Callback does not update entitlements when Shopify status is not active.

## Commit message

Use this commit message for the fix:

```text
fix(billing): derive subscription terms from server catalog
```
