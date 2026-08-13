# Secure Billing Flow Design

## Goal

Fix the pricing and billing security issues without removing any existing plan. The merchant can still choose legacy monthly, legacy annual, Plus, Pro monthly, Pro annual, and partner Pro annual plans. The browser can choose only the plan ID and optional coupon code. The server owns the price, interval, trial, discount, entitlement data, and callback state.

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

The fix is to store a one-time billing state at initiation. The return URL carries only `shop` and `state`. The callback verifies that Shopify activated the expected subscription, consumes the state atomically, and writes entitlements from the stored state, not from query params.

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
8. The endpoint creates a random billing state and passes `shop` and `state` into the Shopify return URL.
9. Shopify receives price, currency, interval, name, and trial days from trusted details only.
10. After Shopify returns an app subscription ID and confirmation URL, the endpoint stores the pending billing state with the expected subscription ID and commercial terms.
11. The merchant approves billing in Shopify.
12. Shopify redirects to `/api/v1/billing/checkSubscriptionStatus?shop=<shop>&state=<token>` and includes its billing identifier.
13. The callback loads the merchant by `shop` and checks `users.acces_checking.pendingBillingStates[state]`.
14. The callback verifies Shopify activated the exact expected subscription from pending state. It must not accept any active charge for the shop.
15. The callback atomically writes `bucks_plan`, `plan_type`, accepted price, trial dates, subscription ID, and the consumed state marker in one DB update.
16. The callback syncs metafields from stored state as a retryable side effect and redirects the merchant back to pricing.

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

## Pricing parity guardrails

This fix must not change the merchant-visible pricing logic. The server catalog exists to protect billing, not to redesign pricing.

Keep these current behaviors:

- Legacy monthly amount remains `BASE_PRICE`, currency `USD`, interval `MONTHLY_INTERVAL`, plan `bucks_premium`, plan type `monthly`.
- Legacy annual compare price remains `COMPARE_PRICE * 12`, interval `ANNUAL_INTERVAL`, and discount percent still comes from `calculateDiscountPricing()` with the same user and partner inputs.
- Legacy annual partner discount still uses `partnerData.discountPercent` when present.
- Legacy annual user discount branches remain the same: new users get `ANNUAL_PLAN_NEW_USER_DISCOUNT`, users with `subscription_id` or current `bucks_premium` get `ANNUAL_PLAN_EXISTING_PLUS_DISCOUNT`, and true free users get `ANNUAL_PLAN_EXISTING_FREE_DISCOUNT`.
- Pro annual standard amount remains `NEW_PRO_ANNUAL_PRICE`; partner Pro annual amount remains `NEW_PRO_ANNUAL_PRICE_PARTNER`; full compare price remains `NEW_PRO_ANNUAL_FULL_PRICE`.
- Partner Pro annual override keeps the existing exception for merchants currently on `bucks_premium_65`.
- Pro monthly coupon still validates with the existing `validateCoupon()` helper against Bucks plan ID `bucks_premium_pro`.

Do not change `calculateDiscountPricing()` as part of this security fix. It is shared with the pricing page display. Server billing should validate catalog amounts before calling it rather than changing the shared helper's behavior.

## Partner Pro annual contract

Partner merchants currently get a server-side Pro annual override in `pages/api/v1/billing/initSubscription.js`: standard Pro annual can become partner Pro annual when `partnerData` exists, except for merchants already on the legacy partner annual plan `bucks_premium_65`.

Keep that runtime behavior. The frontend can keep sending `planId: "pro-annual"` when the merchant clicks the displayed Pro annual card. `initSubscription.js` must canonicalize that to `pro-annual-partner` when `partnerData` exists and `userRecord.bucks_plan !== "bucks_premium_65"`. This keeps Shopify approval aligned with the displayed partner price.

## Coupon contract

The browser sends the coupon code only. It does not decide the discounted amount. The server validates coupons against the resolved catalog plan.

`validateCoupon()` currently expects the Bucks plan ID `bucks_premium_pro`, not the catalog ID `pro-monthly`. Keep that behavior for the minimal fix: call `validateCoupon(plan.bucksPlan, couponCode, validCoupon)`. Only plans with `couponEligible: true` can receive a coupon override.

## Trial policy

Trial days must be resolved server side. The browser must not send trial days.

Rules:

- Existing premium merchant with an active `trialEndDate` keeps the remaining days.
- Merchant with `trialLeft > 0` keeps that preserved trial balance.
- Merchant with no `subscription_id` receives partner trial days or `DEFAULT_TRIAL_DAYS`.
- Merchant that already subscribed and has no remaining trial gets `0`.
- Trial days must be an integer from `0` to `90`.

The server-side resolver must mirror the current frontend flow from `pages/pricing/index.jsx`: check active premium trial first, then preserved `trialLeft`, then brand-new merchant default or partner trial, otherwise `0`.

## Billing state

Use the existing `users.acces_checking` JSON field instead of adding a new Prisma model. Store pending billing attempts by state under `users.acces_checking.pendingBillingStates`. This allows multiple Shopify approval tabs to complete independently without overwriting each other.

```js
{
  pendingBillingStates: {
    [state]: {
      state,
      subscriptionId,
      planId,
      bucksPlan,
      planType,
      name,
      amount,
      currencyCode,
      interval,
      trialDays,
      status: "pending",
      expiresAtMs,
      createdAtMs,
    }
  }
}
```

Use numeric timestamps because `acces_checking` is a Prisma `Json?` field. Store `createdAtMs`, `expiresAtMs`, and `consumedAtMs` as numbers and compare with `Date.now()` in raw Mongo queries.

The initiation route must persist each new state with an atomic `prisma.$runCommandRaw()` `$set` at `acces_checking.pendingBillingStates.<state>`. It must not read, merge, and replace the whole `acces_checking` JSON object, because concurrent billing attempts can otherwise overwrite each other.

The state expires after 30 minutes. The callback applies entitlements and consumes the state in one `prisma.$runCommandRaw()` update so a mid-callback failure cannot strand a paid merchant with a consumed state and no local plan. The raw update must match `myshopify_domain`, `acces_checking.pendingBillingStates.<state>.state`, `acces_checking.pendingBillingStates.<state>.status: "pending"`, and `acces_checking.pendingBillingStates.<state>.expiresAtMs > Date.now()`. It must `$set` both the entitlement fields and `status: "consumed"` plus `consumedAtMs`. If exactly one record is not modified, the callback fails.

`state` is generated with `crypto.randomUUID()`. The callback must validate the UUID format before using it in a dynamic Mongo path. A UUID contains no `.` or `$`, so it is safe for the dynamic key path after validation.

Do not prune stale entries by replacing the whole parent JSON during billing initiation. Cleanup should run as a separate safe maintenance step, or use targeted `$unset` paths for known expired state keys. The hot billing initiation path only adds the new state with atomic `$set`.

## Shopify verification

The callback must verify the specific expected subscription, not just any active charge for the shop.

The callback must still verify:

- Shopify response exists.
- Charge status is `active`.
- The active billing record belongs to the stored shop session.
- The active billing record matches `pendingBillingStates[state].subscriptionId`.
- The local entitlement update uses stored state values, not query values.
- Entitlements and state consumption happen in the same atomic DB update.

If keeping the existing REST charge lookup, compare `gid://shopify/AppSubscription/${charge_id}` with `pendingBillingStates[state].subscriptionId` when Shopify provides `charge_id`. If the formats do not match or Shopify does not return the expected ID, fail closed and log the mismatch. A later improvement can switch this verification to an Admin GraphQL app subscription query, but the minimal fix cannot fall back to "any active charge".

The `shop` query param is only a lookup hint to find the user and offline Shopify session. It is never entitlement truth. The pending state and Shopify verification decide whether local entitlements change.

`saveAppMetafields()` runs only after the local entitlement update succeeds. Metafield sync failure must be logged and treated as retryable repair work. It must not roll back paid entitlements or make the consumed callback replayable.

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
- Callback fails when the active Shopify charge does not match the pending subscription ID.
- Callback fails when the pending state is expired.
- Callback atomically sets entitlements and consumes the pending state in one update.
- Metafield sync failure after entitlement update does not roll back or allow callback replay.
- Starting billing attempt A, then billing attempt B, then approving A still resolves A from its own pending state.
- Partner data plus submitted `pro-annual` creates partner annual trusted details unless the merchant is already on `bucks_premium_65`.
- Pro monthly coupon applies through `validateCoupon(plan.bucksPlan, couponCode, validCoupon)`; non-coupon plans ignore or reject coupon overrides.

## Commit message

Use this commit message for the fix:

```text
fix(billing): derive subscription terms from server catalog
```
