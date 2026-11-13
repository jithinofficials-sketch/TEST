# BUCKS BFCM Campaign Design

Audience: BUCKS product owners and engineers familiar with Shopify app billing who need the agreed campaign rules and integration boundaries.

Status: Commercial rules confirmed; final technical design awaiting review. No campaign code is implemented by this document.

## Agreed Offers

All annual offers target current Pro annual, with $239.88/year as the reference price. Percentages are rounded display values; the annual charges below are authoritative.

| Segment | Annual charge | Displayed discount |
| --- | --- | --- |
| Current Free, first installed before campaign start, never subscribed | $71.96 | 70% |
| New Free merchant, first installed on/after campaign start, never subscribed | $83.93 | 65% |
| Current or former paid merchant, including legacy Plus | $95.90 | 60% |

Paid history takes precedence over installation date. Current annual subscribers, legacy or Pro, receive no campaign offers or banners. Former annual subscribers follow the paid-history rule. Unknown or invalid eligibility data must not unlock the 70% branch.

Eligible merchants selecting Pro monthly receive $9.99/month automatically, presented as "Black Friday deal" and "Offer applied" rather than requiring coupon entry. Do not automatically modify an existing subscription. An already-current Pro monthly card must not invite a duplicate subscription solely to reprice it.

Partner merchants follow the same campaign pricing without stacking; partner trial rules remain. Accepted prices recur on subsequent renewals. Disabling BFCM stops new claims, not existing discounted subscriptions.

## Configuration

- `BFCM_ENABLED=false` by default.
- `BFCM_START_AT` and `BFCM_END_AT` require valid timestamps with explicit timezone offsets.
- An active campaign requires the exact enabled value `true`, valid ordered timestamps, and `start <= now < end`.
- Dates will be configured before launch; no dates are invented in code.
- Read these variables on the server at request time. Do not compile the campaign flag into a public frontend environment constant.

Use a single shared pure offer resolver with explicit configuration and clock inputs. Server-side pages pass the resolved offer to components. The backend independently resolves the offer from trusted merchant data. The client may expire a displayed offer using its supplied end timestamp, but cannot authorize a charge.

## Approaches Considered

1. **Recommended: server configuration plus shared resolver.** Smallest coherent change that keeps SSR presentation and billing consistent and permits future campaign reuse.
2. Set legacy `discountEligible` on Pro. Rejected: legacy plan identities, old thresholds and independent frontend prices would produce incorrect subscriptions and displays.
3. Database-managed campaign administration. Deferred: unnecessary storage, endpoints and management UI for this scope.

Merchant burden: the recommended approach adds no merchant settings. Eligible merchants see the exact annual charge and a direct upgrade action. Existing annual merchants see their existing plan information, without campaign promotion.

## Integration

### Billing

Keep the current trusted catalog and pending-billing flow. Resolve active campaign eligibility before partner plan routing so a direct `pro-annual-partner` request cannot bypass campaign segment pricing. Validate that explicit partner-only requests still require a referral.

Preserve Pro plan identity and entitlement checks. The existing standard Pro annual identity can store a campaign-specific accepted price; do not route 70% through the legacy `bucks_premium_70` identity. `utils/billing/pendingBillingState.js` already stores the trusted amount separately from the plan ID.

Use the existing recurring subscription mechanism. No client amount or coupon can override a resolved campaign price. Include `plan_type` in the trusted merchant projection so legacy `bucks_premium` annual subscriptions are excluded correctly.

Expired campaign submissions should fail with a clear refresh-required response rather than silently charging a higher amount. Carry a non-authoritative campaign ID/quoted amount with campaign submissions, compare against the server-resolved offer, and reject stale or mismatched quotes. Ordinary requests still receive server-side eligibility enforcement.

### Pricing and Dashboard

Use the resolved annual charge for monthly equivalent, reference price, discount badge and savings. Preserve existing annual `acceptedPrice` display. All campaign promotion, including monthly Pro messaging, is suppressed for current annual subscribers.

Recover the dashboard banner presentation from Git history, using `05866e7d` for the BFCM behavior and the last pre-deletion version before `025d1e6a` for later dismissal/layout improvements. Adapt rather than revert whole commits.

Restore artwork/layout, countdown, annual pricing CTA, persistent dismissal and tracking with current repo conventions. Use the campaign-wide expiry, not a new per-merchant database deadline. A campaign-specific dismissal key prevents last year's dismissal from hiding a future campaign. Hide the banner when disabled or expired; preserve the historical expired-support state only while an already-open banner transitions to expiry. An expired CTA must not initiate discounted billing.

No storefront widget code changes or shared widget/admin imports are required.

## Legacy Cleanup Boundary

Retain legacy subscription identities, normalization, entitlement recognition and accepted-price rendering. Audit direct legacy purchase requests and prevent the old 70% endpoint path from bypassing the current campaign rules.

Do not remove all legacy calculation code merely because the card is hidden. First prove which supported legacy purchase flows still depend on it. Remove unused campaign UI flags and obsolete branches only with tests demonstrating no change to supported off-campaign behavior. Existing legacy subscribers must retain recognition and display.

## Verification

Test disabled/missing/malformed configuration; exact start/end boundaries; older/new Free merchants; paid-history precedence; missing install dates; current legacy/Pro annual exclusion; partner routing and direct partner IDs; monthly automatic discount; no coupon stacking; Pro identity; stale quotes; accepted-price persistence; campaign expiry; and off-campaign compatibility.

Component tests cover price agreement, automatic applied state, annual exclusion, countdown expiry and dismissal. Browser verification must cover embedded billing navigation and desktop/mobile banner presentation before release.

Update Sidekick explanations and source citations with the implementation. Follow repository Shopify validation requirements for generated UI code. Do not change GraphQL operations unless required by an identified gap.
