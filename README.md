# Auth Session Hardening Design

## Goal

Fix the reported auth bypass and missing-auth issues by binding merchant SSR access to the verified Shopify request session, not to `context.query.shop` or a stored offline session. Keep the public storefront validation endpoint read-only so unauthenticated storefront traffic cannot trigger Shopify Admin API mutations.

## Audience

- Merchant/admin: embedded app SSR pages must only serialize data for the authenticated merchant shop.
- Staff/admin: full-admin pages must only serialize admin data after the authenticated staff shop is verified with `isSuperAdmin(session.shop)`.
- Shopper/storefront: widget validation can remain public, but it must not perform privileged side effects.

## Current behavior

1. `utils/middleware/isSessionValid.js` reads `context.query.shop`.
2. It calls `sessionHandler.loadSessionWithShop(shop)`.
3. It treats the presence of that stored offline session as proof that the HTTP requester is authenticated.
4. Several SSR pages load merchant data from `context.query.shop` before, or without, verifying the request-bound Shopify session.
5. Admin partner pages authorize from an allowlisted `?shop=` value instead of the requester session.
6. `pages/api/v1/settings/validateUser/index.js` is public but loads the offline session for `shop` and calls `saveAppMetafields()`.

## Issue relevance

### #283 Any stored merchant session is treated as authentication for the requester

Relevant. `utils/middleware/isSessionValid.js` uses `loadSessionWithShop(context.query.shop)`. A stored offline session proves the app has an offline token for that shop; it does not prove the current requester owns that shop session.

This is the root helper flaw. Any page using this helper inherits the bypass unless the helper is changed to use `fetchSession({ req, res })` and compare the authenticated `session.shop` with any requested shop.

### #277 Attacker-controlled shop parameter exposes dashboard data

Relevant. `pages/index.jsx` calls `isShopAvailable(context)` and `isSessionValid(context)` using the same attacker-controlled `context.query.shop`. `isShopAvailable()` loads the merchant record and settings before a trusted tenant identity is established. If session validation fails, the page only sets a client-side redirect flag and can still serialize data in `initialData`.

The dashboard must authenticate first, use `session.shop` as the tenant key, and return a server-side redirect or safe empty props on auth failure.

### #276 Public validation request triggers authenticated Shopify metafield mutation

Relevant. `pages/api/v1/settings/validateUser/index.js` is explicitly documented as public storefront validation, but it loads an offline session and calls `saveAppMetafields(shop, session.accessToken, metaData)` before checking the MD5 `param`.

The endpoint must remain public and read-only. Metafield sync belongs in authenticated settings write flows and billing/status flows, where it already exists.

### #265 Advanced page exposes cross-shop merchant data

Relevant. `pages/advanced/index.jsx` reads `context.query.shop` and calls `loadShopData(shop)` directly. It does not call `isSessionValid`, `isShopAvailable`, `getMerchantPageContext`, or another request-bound auth guard before serializing `settings` and `userData`.

The page must use the same authenticated merchant page context as other merchant SSR pages.

### #264 Partners dashboard trusts allowlisted shop query without authenticating requester

Relevant. `pages/admin/partners/index.jsx` calls `isSuperAdmin(requestedShop)` where `requestedShop` is from the query string. It then validates that same shop through the flawed `isSessionValid` helper. A known allowlisted admin shop domain can therefore expose partner records and admin user data.

Full-admin pages must authenticate the incoming request first and apply `isSuperAdmin()` only to `session.shop`.

### #263 Admin partner edit page authenticates requested shop instead of requesting user

Relevant. `pages/admin/partners/[id].jsx` has the same flaw as #264, then loads a route-selected partner record by `id`.

The same full-admin SSR guard should protect both admin partner pages.

## Related-area audit

I searched SSR pages and session helpers for these patterns:

```bash
rg "getServerSideProps|loadShopData\(|getMerchantPageContext\(|getImpersonatedMerchantPageContext\(|loadSessionWithShop\(|fetchSession\(|context\.query\?\.shop|context\.query\.shop|isSessionValid\(" pages utils
```

### Must fix with this root cause

These areas either serialize merchant/admin data from `context.query.shop`, authorize an admin page from query-string shop, or inherit the broken `isSessionValid()` behavior. They should be changed in the same implementation because the root cause is the same.

- `pages/settings/index.jsx`: loads `loadShopData(context.query.shop)` directly.
- `pages/pricing/index.jsx`: loads `loadShopData(context.query.shop)` directly.
- `pages/partners/index.jsx`: loads `loadShopData(context.query.shop)` directly.
- `pages/currency-rules/index.jsx`: loads `loadShopData(context.query.shop)` directly.
- `pages/analytics/index.jsx`: uses `getMerchantPageContext()`, so it is affected until that helper authenticates before loading data.
- `pages/integrations/index.jsx`: uses `getMerchantPageContext()`, so it is affected until that helper authenticates before loading data.
- `utils/ssr/getMerchantPageContext.js`: currently depends on the flawed `isSessionValid()` implementation.
- `pages/admin/merchants/index.jsx`: uses `fetchSession()`, but falls back to `sessionHandler.loadSessionWithShop(shop)` if request session lookup fails. That fallback recreates the bypass and must be removed.

### Related but already uses a dedicated impersonation guard

These pages use `getImpersonatedMerchantPageContext()`, which calls `getEffectiveShopContext()`. That helper verifies the staff request session with `fetchSession({ req, res })`, checks `isImpersonator(staffShop)`, checks the active impersonation session, then loads merchant data. They should not be changed for this issue unless testing shows the impersonation helper itself is broken.

- `pages/admin/merchants/[merchant]/index.jsx`
- `pages/admin/merchants/[merchant]/settings.jsx`
- `pages/admin/merchants/[merchant]/advanced.jsx`
- `pages/admin/merchants/[merchant]/pricing.jsx`
- `pages/admin/merchants/[merchant]/partners.jsx`
- `pages/admin/merchants/[merchant]/integrations.jsx`
- `pages/admin/merchants/[merchant]/analytics.jsx`
- `pages/admin/merchants/[merchant]/currency-rules.jsx`

### Related but not the same root-cause fix

These files call `loadSessionWithShop(shop)`, but not as SSR requester authentication. They should be documented and regression-tested where relevant, not blindly changed in this auth-hardening patch.

- `pages/api/v1/user/latest.js`: under `withImpersonation()`, it loads the merchant offline session only after the staff session and impersonation session are validated. This is an authorized server-to-Shopify operation and should remain.
- `pages/api/v1/billing/checkSubscriptionStatus.js`: Shopify billing callback uses the shop offline session to verify the charge. It has separate billing-state security concerns, but it is not the same requester-auth bypass covered by these issues.
- `utils/helper.js` `checkMoneyFormat(shop)`: helper loads the offline session for a shop-domain server operation. It is not used in the audited SSR paths; no direct change for this issue.
- `utils/extensionCheck.js`: helper loads the offline session for extension/theme status checks. It is not used in the audited SSR paths; no direct change for this issue.
- `pages/api/v1/public/moneyFormat.js`: public onboarding helper loads a shop offline session and reads Shopify shop money formats. This is a public endpoint with authenticated Admin API read side effects, but it is not in the pasted issues and does not serialize another merchant's app DB data through SSR. Track separately unless the current security batch is explicitly expanded to public endpoint API-capacity hardening.

### Safe redirect-only query usage

- `pages/admin/merchants/list.jsx`: reads `context.query.shop` only to redirect to `/admin/merchants?shop=...`. It does not load merchant data or authorize access. No change needed for this issue.

## Trust model

`context.query.shop` is navigation context only. It can help decide where to redirect, but it must never decide tenant identity or authorization.

The tenant key for merchant pages is:

```text
fetchSession({ req, res }).shop
```

The authorization key for full-admin pages is:

```text
isSuperAdmin(fetchSession({ req, res }).shop)
```

Stored offline sessions loaded with `loadSessionWithShop(shop)` are allowed only after authorization has already established the effective shop. They are valid for server-to-Shopify operations, not for authenticating HTTP requesters.

## Target architecture

### Request session validation

Add a small shared auth utility that performs request-bound validation:

- calls `fetchSession({ req, res })`;
- rejects missing sessions;
- rejects expired sessions when `session.expires` exists;
- normalizes shop domains with trim + lowercase;
- optionally compares authenticated `session.shop` to a requested `shop`;
- returns a structured result containing `session`, `shop`, and failure `reason`.

`isSessionValid(context)` becomes a compatibility wrapper around this utility. It must never import or call `sessionHandler.loadSessionWithShop()`.

### Merchant SSR context

`utils/ssr/getMerchantPageContext.js` should authenticate first:

1. Validate request session with the optional query-shop match.
2. If invalid, return a redirect context with no `user`, `userData`, or `settings`.
3. Use authenticated `session.shop` as the only shop value.
4. Load merchant availability and settings through `isShopAvailable()` using the authenticated shop.
5. Return `superAdmin` and `canImpersonate` from authenticated shop, not query shop.

Every normal merchant SSR page must use the same authenticated source before loading merchant, settings, partner, pricing, or banner data. Required conversions:

- `pages/index.jsx`
- `pages/settings/index.jsx`
- `pages/advanced/index.jsx`
- `pages/pricing/index.jsx`
- `pages/partners/index.jsx`
- `pages/currency-rules/index.jsx`

`pages/analytics/index.jsx` and `pages/integrations/index.jsx` already call `getMerchantPageContext(context)`. They are fixed through the helper once the helper authenticates before loading availability data.

Pages should not independently call `loadShopData(context.query.shop)` unless they are already inside a verified impersonation flow such as `getImpersonatedMerchantPageContext()`.

`isShopAvailable()` must not accidentally preserve the raw-query contract. Choose one implementation contract and test it:

- Preferred: add `isShopAvailableForShop(shop)` and make it accept only an already-authenticated shop string.
- Acceptable: keep `isShopAvailable(context)`, but `getMerchantPageContext()` must construct a new internal context whose `query.shop` is overwritten with authenticated `session.shop` after request validation.

Tests must assert that Prisma reads use authenticated `session.shop`, not incoming `context.query.shop`.

### Full-admin SSR context

Add role-specific admin SSR guards:

- `getAdminPageContext(context)`: for full-admin pages that require `SUPER_ADMIN_DOMAINS`.
- `getImpersonatorAdminPageContext(context)`: for the merchant panel landing page that allows both `SUPER_ADMIN_DOMAINS` and `STAFF_STORES` through `isImpersonator()`.

Both guards must derive staff identity from `fetchSession({ req, res }).shop` before checking roles. Do not use `getAuthorizedAdminShop({ shop: requestedShop })` for SSR authorization.

For full-admin pages, `getAdminPageContext(context)` should:

1. Validate request session with no dependency on `context.query.shop`.
2. Reject unauthenticated requesters.
3. Reject authenticated shops that fail `isSuperAdmin(session.shop)`.
4. Return `{ forbidden, shop, superAdmin, canImpersonate }`.

Use this helper in:

- `pages/admin/partners/index.jsx`
- `pages/admin/partners/[id].jsx`

For the merchant panel landing page, `getImpersonatorAdminPageContext(context)` should:

1. Validate request session with no dependency on `context.query.shop`.
2. Reject unauthenticated requesters.
3. Reject authenticated shops that fail `isImpersonator(session.shop)`.
4. Return `{ forbidden, shop, superAdmin, canImpersonate }`, where `superAdmin` still reflects `isSuperAdmin(session.shop)`.

Use this helper in `pages/admin/merchants/index.jsx` and remove both unsafe patterns there:

- `getAuthorizedAdminShop({ shop: requestedShop })` as SSR authorization.
- fallback to `sessionHandler.loadSessionWithShop(shop)` when request-session lookup fails.

### Public storefront validation

`pages/api/v1/settings/validateUser/index.js` remains public because the widget calls it from storefronts. It should only:

1. accept `GET`;
2. require `shop`;
3. read `settings` and `users` records;
4. return `{ status: "true" }` only when `md5(userData.bucks_plan) === param`;
5. return `{ status: "false" }` otherwise.

It must not load a stored session and must not call `saveAppMetafields()`.

Response contract must preserve storefront widget behavior from `widgets/src/apps/widgets/buckscc/validateUserPlan.js`, where AJAX `error` rejects the promise. For `GET` with a syntactically valid `shop`, return HTTP 200 with `{ status: "true" | "false" }` even when settings or user records are missing. Use 400 only for malformed requests, such as missing/blank `shop`, and 405 for method mismatch.

## Page behavior after fix

Merchant pages with missing or mismatched request sessions should use server-side redirects, not client-only redirect flags that serialize sensitive props first.

Recommended redirect target must match the existing catch-all route `pages/exitframe/[...shop].js`:

```text
/exitframe/<encoded-shop>
```

Use `encodeURIComponent(shop)` when constructing this path. Do not use `/exitframe?shop=<shop>` unless a top-level `/exitframe` route is added and tested. If the code needs to preserve existing auth behavior for unavailable shops, redirect to the existing auth route without including merchant data in props.

## Testing strategy

Add focused Vitest coverage before implementation.

- `tests/utils/auth/requestSession.test.js`: request session success, missing session, expired session, query-shop mismatch, normalized match.
- `tests/utils/middleware/isSessionValid.test.js`: wrapper calls request-bound flow and rejects mismatches without using offline sessions.
- `tests/utils/ssr/getMerchantPageContext.test.js`: no merchant data loaded when session validation fails; authenticated shop is used instead of query shop.
- `tests/utils/admin/getAdminPageContext.test.js`: full-admin access derives from `session.shop`; query-shop allowlist does not grant access; merchant-panel impersonator access derives from `session.shop` and allows `STAFF_STORES` through `isImpersonator()`.
- `tests/pages/merchantSsrAuth.test.js`: each normal merchant SSR entrypoint rejects or ignores mismatched query shop and never loads data for the attacker-selected shop.
- `tests/api/settingsValidateUser.test.js`: endpoint does not call `saveAppMetafields()` or `loadSessionWithShop()` and returns the expected MD5 validation status, including 200 `{ status: "false" }` for missing settings/user data.

Run targeted tests first, then run the full suite:

```bash
yarn test tests/utils/auth/requestSession.test.js
yarn test tests/utils/middleware/isSessionValid.test.js
yarn test tests/utils/ssr/getMerchantPageContext.test.js
yarn test tests/utils/admin/getAdminPageContext.test.js
yarn test tests/pages/merchantSsrAuth.test.js
yarn test tests/api/settingsValidateUser.test.js
yarn test
```

## Out of scope

- Redesigning merchant UI or navigation.
- Changing impersonated merchant route contracts under `pages/admin/merchants/[merchant]/*`, except removing unsafe auth fallback from the merchant panel landing page.
- Adding rate limiting to `validateUser`. It is useful defense-in-depth, but the critical fix is removing public side effects.
- Changing `pages/api/v1/public/moneyFormat.js`. It may deserve a separate public-endpoint hardening issue, but it is outside the reported SSR auth-bypass root cause.
- Changing `pages/api/v1/billing/checkSubscriptionStatus.js`. Billing callback trust was handled by the separate secure billing-flow spec and should not be mixed into this auth SSR patch.
- Changing Shopify metafield payload shape.

## Commit message

Use this commit message after the implementation and tests pass:

```text
fix(auth): bind merchant SSR pages to verified Shopify sessions
```

If the implementation is split into multiple commits, use:

```text
fix(auth): validate SSR sessions from requests
fix(admin): derive partner dashboard access from staff sessions
fix(settings): keep storefront validation endpoint read-only
```
