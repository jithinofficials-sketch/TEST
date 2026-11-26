# RFC: Support-Issued BFCM Discount Offers

- Status: Draft - business decisions pending
- Type: Feature RFC
- Owner: BUCKS team
- Reviewers: Product, Support, Engineering

## 1. Problem

Support needs to give selected merchants an additional BFCM discount after working with them. The offer must be private, attributed to the support member, and usable only by the intended store.

The existing pricing-card coupon displays a public code and supports Pro monthly only. The current BFCM implementation disables coupons and excludes annual subscribers. It does not cover this support flow.

## 2. Why This Matters

A support-issued offer gives the team a way to close upgrades and retain existing Pro annual merchants without publishing a discount for every store.

Measure offers issued, Shopify-approved offers, activated subscriptions, and conversion by support member. A link click alone is not a successful redemption.

## 3. Proposed Solution

Add **Generate BFCM offer link** to the internal merchant-management panel used by support. This is not a setting in the merchant's BUCKS app.

Support generates an offer for the selected store and shares its link. The merchant opens it, sees the eligible discounted plans, and continues to Shopify billing approval.

- Bind each offer to one shop and one campaign.
- Include the support member's name in its code, for example `BFCM-AMRUTHA-7K2P`.
- Record staff identity from the authenticated support account, not from the code text.
- Keep offer rules fixed for this campaign; do not build a general coupon-management screen.
- Do not display the private code publicly on pricing cards.

### Why links from the internal panel?

| Option | Benefit | Tradeoff |
| --- | --- | --- |
| Shared support code entered on pricing cards | Fastest initial build; existing input can be reused | Code can spread; difficult to control individual offers |
| Shop-specific code entered manually | Store restriction and staff attribution | Merchant must copy and enter the code |
| **Shop-specific link generated in the panel** | Support selects the correct store; merchant sees the offer without typing; offer status is trackable | Requires offer storage and a small internal-panel action |

Recommend panel-generated links for this support-only use case. They reduce merchant effort and issuance mistakes. Store binding and backend validation provide the protection, not the link itself. Manual code entry can be deferred; if added, it must use the same offer validation.

### Eligibility

| Current plan | Eligible destination or benefit |
| --- | --- |
| Existing Free | Pro monthly or Pro annual |
| Legacy monthly | Pro monthly or Pro annual |
| Existing Pro monthly | Pro annual only |
| Legacy annual | Pro monthly or Pro annual |
| Existing Pro annual | Discount on the existing Pro annual plan starting next billing cycle |

New Free merchants are outside the proposed eligibility. Existing Pro monthly renewal discounts are also outside it unless Product expands the scope.

Shipping this feature does not automatically alter existing subscriptions. Merchants must redeem an issued offer and complete the required Shopify approval.

## 4. End-to-End Flow

```text
Support opens merchant record
  -> Generates shop-specific offer
  -> Shares link
Merchant opens link and authenticates
  -> Backend validates shop, campaign, eligibility, and price
  -> Pricing page previews the offer
  -> Merchant continues to Shopify approval
  -> Backend verifies Shopify subscription state
  -> UI shows active discount or scheduled renewal discount
```

Opening the link must not create or change a subscription. Preserve the offer through authentication and require a deliberate action to continue to Shopify approval.

### Pricing UI

- **Eligible Pro monthly card:** update the price and show a **Coupon applied** badge after successful validation. Explain the billing amount and discount duration. This badge means the offer is applied to the preview, not that billing is approved.
- **Eligible Pro annual upgrade card:** show the discounted annual charge prominently, with the monthly equivalent secondary. Show when billing starts and how long the discount lasts.
- **Existing Pro annual, after verified approval:** show a **Discount scheduled** badge and a message such as: "From your next billing cycle on [date], your annual payment will be $[amount]."
- If Product approves the 80%-total-discount interpretation, that message can include: "You'll receive 80% off from your next billing cycle." **80% is an unapproved example, not final copy.**
- Before Shopify approval, show **Approval required**, not **Discount scheduled**. If verification is pending, show that status without claiming success.
- Show expired, revoked, wrong-store, or ineligible offers clearly, without silently substituting a different price.

There are no storefront changes.

## 5. Implementation Details

### Offer record and support panel

Store the shop, unique offer identifier/code, campaign, staff identity/name, issue time, optional expiry, eligible destinations, approved discount rule/duration, and status. Link the offer to the Shopify subscription request and its quoted base/final amounts.

Track issued, approval-pending, scheduled, redeemed/active, expired, and revoked states as appropriate. Keep approval distinct from activation. Allow authorized staff to revoke unapproved offers; handling an already approved replacement is a separate billing operation.

### Backend and billing

- Restrict issuance to authorized support staff. Authenticate redemption and match the session shop to the offer.
- Calculate prices on the server. Do not trust discount percentages, staff names, or amounts supplied in a URL or browser request.
- Recheck eligibility and subscription state before creating a billing request. Prevent duplicate requests and repeated discount stacking.
- Reuse the existing subscription creation, pending billing state, callback, and webhook paths, extending them for scheduled replacements.
- For an existing Pro annual renewal offer, create the replacement with `APPLY_ON_NEXT_BILLING_CYCLE` and the same billing currency. Shopify ignores this deferral if the currency changes.
- Do not overwrite the current active plan or accepted price merely because the merchant approved a future replacement. Reconcile activation with Shopify.
- Finalize timing for other plan transitions before implementation; do not apply one replacement behavior to every transition.
- Confirm the Shopify implementation for the approved discount duration. A permanently reduced price must not be used for an offer intended to last only one cycle.
- Revise the existing BFCM annual-exclusion and coupon-disable rules explicitly. Use one shared eligibility/quote calculation for preview and billing.

### Edge cases and verification

| Case | Required behavior |
| --- | --- |
| Link forwarded to another store | Reject redemption for that store |
| Repeated clicks or multiple offers | No duplicate billing replacements or compounded discounts; show existing status |
| Plan or price changes after issuance | Revalidate and require review of any changed quote |
| Approval declined or abandoned | Keep the current subscription; do not mark redeemed |
| Offer expires while Shopify approval is open | Follow the explicit approval-expiry policy agreed below; a pre-redirect check alone is insufficient |
| Shopify request succeeds but local save fails | Reconcile the existing request before retrying; avoid issuing duplicates |
| Delayed callbacks or webhooks | Verify Shopify state and update idempotently; do not claim activation early |
| Cancellation, uninstall, or another upgrade before renewal | Reconcile the scheduled replacement and stop showing an invalid renewal promise |

Verify every eligibility row, price rounding, authorization, wrong-store redemption, duplicate requests, and expiry boundaries. Test immediate and deferred subscription transitions in Shopify test billing, including return-page and webhook handling.

### Delivery

1. Agree on the business questions below; verify renewal and duration behavior with Shopify test billing.
2. Build the fixed-rule offer record, internal-panel action, link redemption, pricing feedback, and billing reconciliation as one end-to-end release.
3. Pilot with selected support staff, then expand after checking approved versus activated offers and billing accuracy.

Manual code entry and broader coupon administration are deferred.

## 6. Open Questions Requiring Approval

### A. What does an additional 10% mean?

Using configured BUCKS defaults, Pro is $19.99/month and the full annual comparison price is $239.88. An existing 70%-off annual offer charges $71.96.

| Interpretation | Next annual charge |
| --- | --- |
| 10% off the current $71.96 payment | **$64.76**, approximately 73% total savings |
| Add 10 percentage points: 70% becomes 80% | **$47.98** |

Recommendation: 10% off the actual eligible price. Product must approve the interpretation before prices or percentage copy are implemented. Existing subscriptions use the verified accepted price, not a percentage reconstructed from a rounded badge.

### B. Does the offer expire?

Recommendation: yes. Decide the redemption window, campaign cutoff, and exact timezone. Separately decide whether Shopify approval must finish before expiry and how outstanding approval requests are handled.

An offer approved within its valid window should retain its promised next-cycle benefit even when the renewal date falls after campaign expiry.

### C. How long does the discount last?

Choose one billing cycle, a fixed number of cycles, or every renewal. "Starting next billing cycle" does not answer this. UI copy and Shopify billing configuration must match the decision.

### D. Which price is discounted, and what can stack?

For upgrades, decide whether the base is the regular destination price, BFCM price, or partner price. For existing Pro annual renewals, recommend the actual accepted annual charge. Apply the support discount once and define how existing coupons interact with it.

### E. When do other plan transitions take effect?

Confirm timing for monthly-to-annual, legacy annual-to-Pro annual, and legacy annual-to-Pro monthly transitions. Show the approved timing and Shopify billing terms before confirmation.

### F. Operational limits

Confirm which staff roles can issue offers, how competing offers for the same store are handled, and the definition of "existing Free" at the campaign cutoff. Recommendation: one redeemable offer per shop per campaign, with retries reopening the same offer rather than issuing another discount.

## References

- `pages/pricing/index.jsx`: plan eligibility layout, pricing display, BFCM/coupon flags.
- `components/pricing/PricingCard.jsx`: existing coupon UI.
- `utils/billing/bfcmCampaign.js`: current BFCM eligibility and quotes.
- `pages/api/v1/billing/initSubscription.js`: trusted pricing and subscription initiation.
- `utils/billing/newPlanHelpers.js`: current coupon validation and immediate replacement default.
- `utils/common/constants/pricingConstants.js`: default prices and BFCM amounts.
- [Shopify subscription replacement behavior](https://shopify.dev/docs/api/admin-graphql/latest/enums/AppSubscriptionReplacementBehavior).
