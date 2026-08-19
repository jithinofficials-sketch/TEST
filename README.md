# Route-Scoped CORS Design Spec

**Date:** 2026-08-19
**Status:** Draft for implementation
**Scope:** Replace global wildcard CORS with a centralized route-aware policy for Bucks admin APIs, public widget APIs, Shopify app proxy routes, and webhook routes.

## Problem

Bucks currently emits wildcard browser CORS headers too broadly:

- `next.config.js:50` applies `Access-Control-Allow-Origin: *` to every `/api/:path*` route.
- `middleware.js:26` applies `Access-Control-Allow-Origin: *` to matched app routes.

This exposes API responses to any browser origin that can trigger a request and receive a readable response. The audit did not find credential reflection, but the global wildcard API exposure is confirmed.

The fix must remove the global wildcard and replace it with route-scoped origin decisions. Authentication, Shopify proxy signatures, and webhook HMAC verification remain the real access-control boundaries; CORS only controls which browser origins can read responses.

## Audience

This affects both Bucks audiences:

- **Merchant admin:** embedded Shopify Admin pages and admin API calls must keep working only from expected Shopify/app origins.
- **Shopper storefront:** the storefront widget must keep reading public widget config/rates from legitimate storefront origins without opening protected APIs to arbitrary sites.

If merchant and shopper needs conflict, protected merchant/admin data wins. Public widget data may be readable cross-origin only where the widget requires it.

## Goals

- Remove wildcard CORS from global Next headers and app middleware.
- Centralize CORS policy so future API routes do not copy/paste ad hoc header logic.
- Allow protected admin/app APIs only from expected Shopify Admin, app-host, development tunnel, and valid Shopify shop origins.
- Allow public storefront/widget APIs from valid Shopify storefront origins and a documented custom storefront allowlist.
- Keep Shopify app proxy routes protected by `verifyProxy`; CORS must not replace proxy signature verification.
- Keep webhook/GDPR routes protected by Shopify signature/HMAC verification; they must not rely on browser CORS.
- Add tests proving `Origin: https://evil.example` is rejected for protected route families.
- Add tests proving wildcard `Access-Control-Allow-Origin: *` no longer appears in project-owned CORS code.

## Non-Goals

- Do not redesign API authentication.
- Do not change Shopify OAuth behavior.
- Do not add a database-backed custom domain lookup in this fix.
- Do not modify widget JavaScript bundles under `widgets/`.
- Do not add a third-party CORS package.
- Do not make webhook endpoints inaccessible to Shopify server-to-server delivery.

## Route Families

### Protected Admin/App APIs

Examples:

- `/api/v1/user/*`
- `/api/v1/settings/*`
- `/api/v1/billing/*`
- `/api/v1/banner/*`
- `/api/v1/analytics/*`
- `/api/v1/admin/*`
- `/api/partner/*`
- `/api/v1/public/themeAppEmbeds`

Policy:

- Allow browser CORS only from expected admin/app origins.
- Reject preflight requests from unknown origins with `403`.
- Do not set an allow-origin header for unknown normal requests.
- Keep existing session and impersonation checks unchanged.

`/api/v1/public/themeAppEmbeds` is classified as admin despite the `/public/` path segment because it calls `fetchSession()` and is consumed by admin onboarding UI, not by the storefront widget runtime.

Allowed origins:

- `https://admin.shopify.com`
- `process.env.SHOPIFY_APP_URL`, if valid
- `process.env.TUNNEL_URL`, in development only, if valid
- `https://{shop}` when `{shop}` is a valid `*.myshopify.com` domain available from `req.query.shop`, `req.body.shop`, `req.shop`, or the current session context

### Public Storefront/Widget APIs

Examples:

- `/api/v1/public/config`
- `/api/v1/public/currency`
- `/api/v1/public/appStatus`
- `/api/v1/public/moneyFormat`
- `/api/v1/settings/validateUser`

Policy:

- Allow browser CORS only where the storefront widget needs cross-origin reads.
- Allow valid Shopify-hosted storefront origins: `https://*.myshopify.com`.
- Allow custom storefront domains only through an explicit environment allowlist.
- Reject preflight requests from unknown origins with `403`.
- Do not use `*` as a fallback.

Custom domain allowlist:

- Add `CORS_STOREFRONT_ORIGINS` as a comma-separated list of exact origins.
- Example: `https://example.com,https://www.example.com`.
- This is intentionally explicit until Bucks has a reliable domain-to-shop lookup in the database or via Shopify Admin API.

Reason: allowing every arbitrary custom domain recreates the wildcard exposure under a different name. A future domain lookup can replace the env allowlist when the app stores or verifies merchant domains reliably.

`/api/v1/settings/validateUser` is intentionally included in the public-widget family even though it lives under `/api/v1/settings`. The route file documents it as a non-impersonatable public storefront endpoint, and the widget calls it from `widgets/src/apps/widgets/buckscc/validateUserPlan.js`. Route classification must check this exact path before the broad `/api/v1/settings` admin prefix.

### Shopify App Proxy APIs

Example:

- `/api/proxy_route/*`

Policy:

- Keep `verifyProxy` as the access-control boundary.
- CORS may allow valid Shopify storefront origins if browser reads are required.
- Unknown origins do not receive CORS allow headers.

### Auth/OAuth APIs

Examples:

- `/api/auth/*`

Policy:

- Do not add API CORS headers.
- These routes are browser navigations and redirects, not JSON APIs consumed by arbitrary browser JavaScript.

### Webhooks and GDPR Hooks

Examples:

- `/api/webhooks`
- `/api/webhooks/*`
- `/api/v1/hooks/*`
- `/api/gdpr/*`

Policy:

- Do not add CORS allow headers.
- Continue to rely on Shopify webhook processing and HMAC/signature middleware.
- `OPTIONS` requests should not be granted cross-origin access.

## Architecture

Add one server-side CORS module at `utils/security/cors.js`.

The module owns:

- route-family classification from `req.url`, with exact public-widget exceptions checked before broad admin prefixes
- origin parsing and normalization
- allowlist checks
- CORS response headers
- preflight responses
- a wrapper for API route handlers

Proposed exports:

- `CORS_POLICIES`
- `getCorsPolicyForPath(pathname)`
- `isOriginAllowedForPolicy(origin, policyName, req)`
- `applyCorsHeaders(req, res, policyName)`
- `withCors(policyNameOrHandler?)`

`withCors()` should run before route logic. For valid preflight requests it returns `204`. For invalid preflight requests it returns `403`. For normal requests it sets CORS headers only when the origin is allowed, then calls the wrapped handler.

## Header Contract

For an allowed origin:

- `Access-Control-Allow-Origin: <request origin>`
- `Vary: Origin`
- `Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS`
- `Access-Control-Allow-Headers: Content-Type, Authorization, X-Impersonating-Shop, X-Impersonation-Session`

For a rejected origin:

- no `Access-Control-Allow-Origin`
- preflight returns `403`
- normal requests continue to handler authentication unless route-specific auth rejects them first

Do not set `Access-Control-Allow-Origin: *` anywhere.

Do not set `Access-Control-Allow-Credentials` unless a later route proves it needs credentialed cross-origin browser requests. Current acceptance criteria do not require it.

## Implementation Application

Apply the wrapper to route families that need explicit CORS behavior first:

- `pages/api/v1/settings/index.js` as the protected admin smoke route.
- `pages/api/v1/user/latest.js` as another protected admin route with Shopify Admin API reads.
- `pages/api/v1/billing/initSubscription.js` as a protected billing route.
- `pages/api/v1/public/config.js` as the public widget config smoke route.
- `pages/api/v1/public/currency.js` as public rates data.
- `pages/api/v1/public/appStatus.js` as public app status data.
- `pages/api/v1/public/moneyFormat.js` as public onboarding/widget support data.
- `pages/api/v1/settings/validateUser/index.js` as public storefront plan validation; wrap with `public-widget`, not `admin`.
- `pages/api/v1/public/themeAppEmbeds.js` as an admin/session-bound route; wrap with `admin` if it is wrapped at all.
- `pages/api/proxy_route/json.js` as app-proxy CORS behavior while keeping `verifyProxy`.

Then add `withCors` to additional protected API routes in the same families. If the first implementation needs to stay small, cover all `/api/v1/public/*` and at least one representative protected route with tests, then follow up with full route-family wrapping before merging.

## Error Handling

- Invalid `Origin` value: treat as rejected.
- Missing `Origin`: do not set CORS headers and do not reject; non-browser and same-origin requests should continue to existing auth/handler logic.
- Invalid preflight from unknown origin: return `403` JSON `{ success: false, error: "Origin not allowed" }` or plain end body; tests should assert the status and missing wildcard, not the exact copy.
- Allowed preflight: return `204` with CORS headers.

## Testing

Add unit tests for `utils/security/cors.js`:

- protected admin route rejects `https://evil.example`
- protected admin route allows `https://admin.shopify.com`
- protected admin route allows configured `SHOPIFY_APP_URL`
- public widget route allows `https://test-shop.myshopify.com`
- public widget route allows an exact `CORS_STOREFRONT_ORIGINS` match
- public widget validation route `/api/v1/settings/validateUser` allows `https://test-shop.myshopify.com`
- public widget validation route rejects `https://evil.example`
- public widget route rejects `https://evil.example`
- webhook route policy does not allow browser CORS
- rejected origins never emit `Access-Control-Allow-Origin: *`

Add route-level smoke tests:

- `OPTIONS /api/v1/settings` with `Origin: https://evil.example` returns `403` and no allow-origin header.
- `OPTIONS /api/v1/public/config` with `Origin: https://test-shop.myshopify.com` returns `204` and reflects that origin.
- `OPTIONS /api/v1/settings/validateUser?shop=test-shop.myshopify.com&param=hash` with `Origin: https://test-shop.myshopify.com` returns `204` and reflects that origin.
- `OPTIONS /api/v1/settings/validateUser?shop=test-shop.myshopify.com&param=hash` with `Origin: https://evil.example` returns `403` and no allow-origin header.
- `OPTIONS /api/webhooks/app_uninstalled` with `Origin: https://evil.example` does not return wildcard CORS.

Add static regression tests:

- `next.config.js` does not contain `Access-Control-Allow-Origin`.
- `middleware.js` does not contain `Access-Control-Allow-Origin`.
- project source does not contain `Access-Control-Allow-Origin', '*'`, `Access-Control-Allow-Origin", "*"`, or equivalent project-owned wildcard CORS patterns.

## Acceptance Criteria Mapping

- **Wildcard CORS removed from global Next headers and middleware:** remove from `next.config.js` and `middleware.js`; static tests cover this.
- **Admin/API routes only allow expected Shopify/admin application origins:** `admin` policy covers Shopify Admin, app host, dev tunnel, and valid shop origins.
- **Public storefront/widget routes have documented narrow policy:** `public-widget` allows `*.myshopify.com` plus explicit `CORS_STOREFRONT_ORIGINS` exact matches.
- **Storefront validation remains public:** `/api/v1/settings/validateUser` is classified as `public-widget` before `/api/v1/settings/*` is classified as `admin`.
- **Webhook routes do not rely on browser CORS:** webhook routes get no CORS allow headers; Shopify HMAC/signature verification remains unchanged.
- **Smoke checks reject evil origin:** route-level tests cover protected and public route families.

## Risks

- Some merchants use custom storefront domains. Exact env allowlisting is safe but operationally limited. A follow-up can add a verified domain lookup through Shopify Admin API or persisted shop domains.
- Removing wildcard CORS before production `CORS_STOREFRONT_ORIGINS` is populated can break widget config reads on custom storefront domains. Production rollout must inventory and configure known custom storefront origins, or defer enforcement until verified shop-domain lookup exists.
- Admin APIs may be called from `https://admin.shopify.com` with session tokens or from the app host depending on browser/embed context. Tests must cover both configured app host and Shopify Admin origin.
- Normal requests with disallowed origins still execute handler logic if they are not preflight. This is acceptable because CORS is not authentication; session/proxy/HMAC checks remain required. The browser will not expose the response without an allow-origin header.

## Open Decisions

None for the initial implementation. The chosen public storefront policy is Shopify storefronts plus an explicit custom-domain env allowlist.
