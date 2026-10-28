# Merchant Panel Enhancement Design

Status: approved scope; written design for review. Implementation has not started.

Audience: Bucks internal super-admin and allowlisted support staff.

## Decisions

The RFC and clarification answers approve Polaris Filters and IndexTable, complete-dataset filtering, per-row Sync, install-time Markets capture, and a Live Preview button in each merchant row. External feedback investigation/restoration is a separate follow-up, with no rating column in this release.

Use the existing shop-level enabled presentment currencies, refreshed manually through Sync. Use UTC date boundaries and single-select Bucks Plan. Sync does not reconcile billing. The later clarification approves displaying the stored Bucks plan with recorded subscription status as secondary context: missing status must not turn a paid plan into Free.

## Approach

Implement focused changes to the existing list component and route, plus small helpers for query validation, row serialization, and Shopify context refresh. Reuse the refresh helper from installation and Sync. No new UI library, global store slice, widget changes, or general-purpose data framework.

Alternatives: a display-only patch is smaller but cannot satisfy reliable filtering or refresh; automatic Markets/currency webhook refresh adds integration and rollout work that is unnecessary for the approved manual-refresh scope. The focused approach meets the agreed scope with the least additional surface. Staff get one list and explicit refresh; merchants gain no settings or setup tasks.

## Interface

Keep the existing Merchants page and Access navigation. Replace the custom table and pagination controls with Polaris components. Use a non-selectable IndexTable, horizontally scrollable on narrow screens.

Columns, in order: Merchant, Email, Domain, Status, Bucks Plan, Store created, App installed, Markets status, Enabled currencies, Actions.

- Merchant: name, falling back to Shopify domain. Email and unavailable dates: N/A.
- Domain: primary domain, falling back to Shopify domain.
- Status: Active or Uninstalled; this describes installation only.
- Bucks Plan: named stored Bucks entitlement, never Shopify plan_display_name. Show recorded subscription status below it when present. Explain that Sync does not verify billing. Known free/null/missing plan maps to Free; an unrecognized nonempty plan maps to Unknown rather than Free. Keep legacy variants distinguishable by label and identifier.
- Store created: created_at from Shopify shop data. App installed: first_installed, never last_installed.
- Markets status: Enabled, Disabled, or Not available. Help text: "Enabled means at least one active market beyond the primary market. Saved data is refreshed on installation or Sync."
- Enabled currencies: saved ISO codes, allowed to wrap. Nonempty historical values display immediately. Empty values without evidence of successful retrieval display Not available; a successfully refreshed empty list displays None. A separate successful currency-refresh timestamp establishes that evidence.
- Actions: Access, Sync, Live Preview. Only the affected row shows Sync loading; Access keeps its existing behavior.

Live Preview is an external-link button with target="_blank" and rel="noopener noreferrer". Build an HTTPS storefront URL from a validated primary hostname, otherwise a validated myshopify.com hostname. Reject credentials, unsafe schemes, malformed hosts, and unexpected ports. Disable the action if neither source is valid. It does not require impersonation and remains usable for an uninstalled merchant. This is the live storefront, subject to password protection.

## Filters and list contract

GET /api/v1/admin/merchants/list accepts page, limit, search, status, plan, createdFrom, createdTo, installedFrom, installedTo. Defaults: page 1, limit 20, no filters. Cap limit at 100. Reject malformed, repeated/array-valued, fractional, nonpositive, or unsafe pagination values with 400; validate skip arithmetic. Search is trimmed and limited to 200 characters.

Status: any, active, uninstalled. Plan: any, free, unknown, or a recognized Bucks plan identifier. Text search matches name, email, primary domain, or Shopify domain with OR. Combine text search, other filters, and staff-own-store exclusion with AND. Use exactly the same predicate for findMany and count before skip/take. Use last_used descending plus myshopify_domain ascending for deterministic ties.

Date inputs accept real YYYY-MM-DD calendar dates. Either bound may be supplied. Start is inclusive at 00:00 UTC; end is exclusive at the next day's 00:00 UTC. Reject reversed or invalid ranges. Missing dates do not match a date filter. Account explicitly for MongoDB missing versus null fields in Free/Unknown plan predicates.

Each date filter opens its own popover containing range controls and Apply. Do not leave calendars permanently visible. Committed changes reset page to 1. Show removable chips and Clear all. Debounce text input by 300 ms; never issue an immediate request for the previous page when a filter changes.

Return a whitelist of display fields, normalized plan metadata, safe storefront URL, and existing pagination shape. Never return tokens or full merchant records. Loading, no results, and request failure have distinct states; errors have Retry. Guard list requests by monotonically increasing request IDs and invalidate on cleanup so old results/errors/finalizers cannot replace current state. After Sync, refetch the current filtered page because changed data may affect membership; if the page is beyond the new total, load the last available page.

## Persistence and refresh

Add nullable users fields:

```prisma
marketsEnabled        Boolean?
marketsFetchedAt      DateTime?
currenciesFetchedAt   DateTime?
```

No default false and no bulk backfill. Missing Markets fields mean Not available. Existing enabled_presentment_currencies remains the currency source. A successful Markets refresh writes its boolean and timestamp together. A successful currency refresh writes its array and timestamp together.

POST /api/v1/admin/merchants/sync accepts only merchantShop as the target selector. Authenticate the actual staff session with fetchSession, authorize with isImpersonator, validate the Shopify hostname, and load the selected merchant and its credentials server-side. Do not use the staff token or create an impersonation session to perform Sync. Return 401 for missing authentication, 403 for unauthorized staff, 405 for wrong method, and 400 for invalid/unknown target or unavailable credentials/uninstalled app.

Refresh has independent shop/currency and Markets outcomes. The shop query selects only name, email, primary domain, creation time, and enabledPresentmentCurrencies. Write only successfully validated fields; never spread Shopify responses or an old complete user record into an update. Preserve first_installed, last_installed, last_used, billing, installation status, settings, and unrelated shop fields.

Use the configured Shopify client and selected merchant credentials. Check transport errors, GraphQL errors, null/missing fields, pagination cursors, and query completeness. A failed Markets request preserves both saved value and successful timestamp. Disabled requires a complete successful traversal with no active non-primary market. Enabled requires a verified active non-primary market; prefer completing traversal before saving for a simple consistent contract. Bound execution time; exhausted pagination/time/retry budgets produce failure, never Disabled.

Return component outcomes, for example data.outcomes.store/currencies/markets with updated or failed and a safe message, plus data.partial. Overall success is true only if something was saved; partial success is visible as a warning rather than a full-success toast. Persistence failure is not reported as successful refresh. Disable repeated row clicks while pending; use bounded retries only for transient reads, respecting Shopify throttle guidance. Do not introduce a distributed job system.

After new-install/reinstall merchant persistence in afterAuth, attempt the same Markets refresh using the authenticated session. Await a bounded attempt within its own error boundary: failures must not fail authentication, roll back merchant creation, or skip existing notifications. Do not launch an unawaited promise whose execution depends on serverless process survival. Do not refresh Markets on every ordinary auth callback for an already active merchant.

## Verified code facts and compatibility gates

- pages/api/auth/afterAuth.js maps REST shop data into users and sets first_installed only on create. Preserve this behavior.
- utils/constants.js excludes enabled_presentment_currencies from ordinary update writes. Sync must use an explicit narrow write, not weaken the global allowlist.
- utils/admin/isSuperAdmin.js authorizes merchant-panel staff through isImpersonator, combining SUPER_ADMIN_DOMAINS and STAFF_STORES.
- Billing confirmation writes bucks_plan and subscription_id, not subscription_status. The local payment_update.js handler is a stub. New webhook registrations target a separate server. Stored status is context, not a verified live billing signal.
- utils/shopify.js reads SHOPIFY_API_VERSION and SCOPES from environment. Deployed values and granted target-store scopes have not been verified.
- Shopify current Market documentation requires read_markets and deprecates primary/enabled fields. Before implementing the query, verify the deployed-version schema and primary-market semantics on a test shop. Do not silently upgrade the app-wide API version or assume modern retail/company markets represent international regions.
- Verify shop.enabledPresentmentCurrencies, primaryDomain, and createdAt against that same version. If existing scopes do not cover Markets, missing permission stays Not available and Sync explains it; any scope rollout requires an explicit deployment decision.

These API checks are blocking implementation prerequisites for the refresh adapter, not claims of completed live validation. The table/filter work is independent.

## Verification and release

Test complete-dataset filter/count parity, Mongo null/missing plan cases, UTC/leap-day boundaries, reinstall preservation, staff authorization, target credential isolation, complete pagination, partial GraphQL errors, missing scopes, revoked credentials, and last-known-good preservation. UI tests cover stale requests, chip removal/reset, row loading, Sync-driven refetch, and safe external links.

On a development shop, verify first installation and reinstall, a primary-only configuration, an active non-primary market, disabled non-primary markets, and partial failures. Check desktop and narrow-screen scrolling, keyboard use, and visible error states. Never declare deployed Markets compatibility based only on mocks.

Use nullable additive schema fields and regenerate Prisma. No destructive data migration or all-store backfill. The storefront widget build remains isolated. Sidekick explanations need changes only if inspection finds an existing explanation of this internal staff surface; do not expose staff-only actions as merchant capabilities.

## References

- components/admin/MerchantList.jsx
- pages/api/v1/admin/merchants/list.js
- components/admin/StaffAccessBar.jsx
- pages/api/auth/afterAuth.js
- utils/userWriteFilter.js and utils/constants.js
- utils/admin/isSuperAdmin.js and utils/admin/impersonationContext.js
- pages/api/v1/billing/checkSubscriptionStatus.js
- utils/common/common-methods/trialHelpers.js
- utils/webhooks/payment_update.js and utils/shopify.js
- https://shopify.dev/docs/api/admin-graphql/latest/objects/Market (reviewed 2026-10-28; not proof of deployed-version compatibility)
