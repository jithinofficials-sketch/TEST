# Comprehensive Performance & Quality Analysis

---

## 1. Already Fixed (This Session)

| # | Issue | Fix | Impact |
|---|-------|-----|--------|
| 1 | No SSR timing visibility | `utils/debugTiming.js` — `startDebugTimer`, `timeAsync`, gated by `BUCKS_DEBUG_TIMING=1` | Debugging |
| 2 | Currency rules data fetched fresh on every SSR (~451ms) | `utils/currencyData.js` — 1h TTL in-memory cache + stale-while-fetch | ~451ms → ~1ms on repeat |
| 3 | Partner data queried from DB on every SSR (~353ms) | `utils/partners/getActivePartners.js` — 5min TTL cache + `invalidatePartnerCache()` | ~353ms → ~1-2ms on repeat |
| 4 | Duplicate sequential DB calls on every SSR (6 queries per page) | `utils/ssr/getMerchantPageContext.js` — parallel `isShopAvailable` + `isSessionValid`; `loadShopData.js` — parallel `getAppDetails` + `getUserDetails` | ~91ms → ~51-67ms |
| 5 | No shared SSR pattern for impersonated merchant pages | `utils/ssr/getImpersonatedMerchantPageContext.js` — security gate + identity split | Correctness + maintainability |
| 6 | Dashboard SSR loading all below-fold widgets synchronously | `ssr: false` + `DashboardWidgetPlaceholder` for 7 components | Faster LCP |
| 7 | Slow public API endpoints with no internal timing | Added timing to `appStatus`, `moneyFormat`, `themeAppEmbeds` | Debugging |
| 8 | `useFetch.js` — forced await on non-promise | Cleaned up | Correctness |
| 9 | `helper.js` — import/export name mismatches | Aligned names | Correctness |
| 10 | `getEffectiveShopContext.js` — unawaited DB call | Added `await` | Correctness |

---

## 2. High Priority Issues (Impact Score: 8-10)

### 2.1 Duplicate DB reads across SSR chain — systemic

**Severity: Critical (duplicates ~4 of 6 DB queries per request)**

The SSR pipeline does the same `settings` + `users` queries twice:
1. `getMerchantPageContext` → `isShopAvailable` → `settings.findUnique` + `users.findUnique`
2. `loadShopData` → `getAppDetails` → `settings.findUnique` again + `getUserDetails` → `users.findUnique` again

**Fix**: Implement a request-scoped in-memory cache (e.g., `Map<shop, Promise<...>>` with auto-cleanup) so `prisma.settings.findUnique({ where: { shop } })` with the same args during one request resolves from cache.

**Impact**: Eliminates ~50-100ms of duplicate DB time per SSR.

### 2.2 `POST /api/v1/public/config.js` + `appStatus.js` — no caching, widget hotpath

**Severity: Critical (called on every storefront page load)**

These endpoints hit MongoDB on every widget page load for every store visitor. `config.js` returns ALL settings fields (~50) when the widget likely needs only 5-10.

**Fix**: Add 5-60s TTL in-memory cache + `select` projection to return only needed fields.

### 2.3 `POST /api/v1/public/moneyFormat.js` — Shopify REST call (~1s+), no cache

**Severity: Critical**

Every widget page load hits the Shopify REST `shop` endpoint (~1s) for `money_format` + `money_with_currency_format`. Only 2 fields needed from a ~50-field response.

**Fix**: Per-shop TTL cache (60s) + `select` projection client-side.

### 2.4 `GET /api/v1/public/themeAppEmbeds.js` — 2 sequential Shopify REST calls (~2s+)

**Severity: High**

Two Shopify REST calls (`themes` + `settings_data.json` ~500KB) run **sequentially** and could run in **parallel**. Also no caching, and fetches entire `settings_data.json` just for one boolean.

**Fix**: Parallelize the two REST calls + per-shop TTL cache (300s) + extract only the single boolean needed from the response.

### 2.5 `sessionHandler.storeSession` — `expires` field never persisted

**Severity: High (session expiry check is unreliable)**

`storeSession`'s upsert (lines 14-28) does not store `session.expires`. `isSessionValid` calls `new Date(session?.expires) < new Date()` which checks a field that's always `undefined` in persisted sessions. This means sessions never appear expired from DB data alone.

**Fix**: Add `expires: session.expires` to both `update` and `create` blocks of the upsert.

### 2.6 `@sentry/browser` — static imports in 3 components (~80KB in critical bundle)

**Severity: High**

`PartnerPromoBanner.jsx`, `AnalyticsBanner/index.jsx`, and `pages/analytics/index.jsx` import Sentry eagerly. Only `onBoardingCard.jsx` does it correctly with a dynamic `import()`.

**Fix**: Convert all Sentry imports to dynamic. Saves ~80KB gzipped from critical path.

### 2.7 `react-query` — installed but never used

**Severity: High (dead ~13KB dependency + manual fetch everywhere)**

`react-query@3.39.3` exists in `package.json` and QueryClient is created in `_app.js`, but ZERO components use it. Manual `useFetch` hooks with no caching, dedup, or retry are used instead.

**Fix**: Either remove it (saves bundle), or migrate analytics page + widget data fetching to React Query (saves boilerplate + adds caching).

### 2.8 `POST /api/v1/user/dashboardSection.js` — NoSQL injection vulnerability

**Severity: Critical (security)**

```js
data: { [section]: value }  // section comes from req.body unfiltered
```
An attacker can set arbitrary user document fields.

**Fix**: Whitelist allowed `section` values.

### 2.9 No rate limiting on any public endpoint

**Severity: High (no protection against traffic spikes/abuse)**

All `/api/v1/public/*` endpoints are internet-facing with zero rate limiting.

**Fix**: Add in-memory rate limiter (e.g., sliding window per IP + per shop) to the public endpoints.

---

## 3. Medium Priority Issues (Impact Score: 5-7)

### 3.1 Admin import runs sequentially before data fetch on 6 pages

`pages/index.jsx`, `settings`, `pricing`, `currency-rules`, `advanced`, `partners` all do:
```js
const admin = await import("@/utils/admin/isSuperAdmin");
// ... check admin sync ...
const data = await loadShopData(shop); // runs second
```

**Fix**: Wrap admin import + data fetch in `Promise.all`. Saves ~50-100ms on first load.

### 3.2 Onboarding step conversion logic duplicated across 2 pages (~38 lines each)

`pages/settings/index.jsx` lines 43-80 and `pages/advanced/index.jsx` lines 48-86 are near-identical `onBoardDbConversion` + `bucksEmbedAppCheck` + `moneyFormatStatus` mutations.

**Fix**: Extract to `syncOnboardingProgress(userData, settings, extensionStatus, moneyFormat)`.

### 3.3 Pricing redirect date inconsistency

`pages/settings/index.jsx` uses `2025-03-07T03:05:00.000Z` while all other pages use `2025-03-06T03:05:00.000Z`. Likely a bug causing settings page to apply the redirect check for an extra day.

**Fix**: Standardize to one date across all pages.

### 3.4 Pricing redirect check duplicated across 5 pages

Same `!userData?.bucks_plan && last_installed > 2025-03-06` logic copy-pasted in `index.jsx`, `settings`, `advanced`, `currency-rules`, `analytics`.

**Fix**: Extract to `checkPricingRedirect(userData)`.

### 3.5 `pages/partners/index.jsx` — fetch data potentially unused by component

Calls `loadShopData(shop)` which fetches `settings` + `userData`, but the component only destructures `superAdmin`/`canImpersonate` from `initialData`.

**Fix**: Verify data usage. If unused, skip the DB query entirely.

### 3.6 `themeAppEmbeds.js` line 54 — typo `trialStartDat` instead of `trialStartDate`

**Fix**: Fix the typo so the field serializes correctly.

### 3.7 `pages/index.jsx` — not using `getMerchantPageContext`

The dashboard (highest-traffic page) has ~100 lines of inline SSR logic duplicating exactly what `getMerchantPageContext` provides.

**Fix**: Migrate to `getMerchantPageContext` and add partner banner fetches in a parallel batch.

### 3.8 `helper.js` — `filterDbData` mutates caller's object

```js
keysToRemove.forEach(key => delete data[key]);  // mutates input!
// then builds filtered copy via .filter()  // renders the delete loop pointless
```

**Fix**: Remove the mutation loop — only the `.filter()` is needed.

### 3.9 `getEffectiveShopContext.js` — two sequential `users` queries instead of one

`getUserDetails(merchant)` (excludes `accessToken`) → then `prisma.users.findUnique({ accessToken: true })` as a separate query.

**Fix**: Merge into a single query that includes `accessToken`, then strip before returning.

### 3.10 `pages/admin/partners/index.jsx` and `[id].jsx` — no timing instrumentation

The only pages completely missing debug timing.

**Fix**: Add `startDebugTimer` + `timeAsync`.

### 3.11 Recharts (~50KB) — analytics charts dynamically imported (good) but charts themselves re-render unnecessarily

`ConversionTrendsChart.jsx` and `RevenueByCurrencyChart.jsx` have no `React.memo`. They re-render on every parent update even with the same `data` prop.

**Fix**: Add `React.memo` to both chart components.

### 3.12 `TourGuide` — statically imported on analytics page, used conditionally

`import TourGuide` at top of `pages/analytics/index.jsx`, but only rendered when `runTour && !suggestionsLoading` (most users see it once or never).

**Fix**: Dynamically import with `dynamic(() => import(...))`.

### 3.13 `_app.js` line 30 — orphan `console.log(test)`

**Fix**: Remove the debug log.

### 3.14 `deleteSession` uses `deleteMany` on unique field

`sessionHandler.js` line 75: `deleteMany({ where: { id } })` — `id` is likely unique. Should use `delete`.

### 3.15 `isSuperAdmin.js` — env vars re-parsed on every function call

`parseShopList(process.env.SUPER_ADMIN_DOMAINS)` called every time `isSuperAdmin`, `getImpersonatorStores`, or `isImpersonator` is invoked. Env vars are static at runtime.

**Fix**: Parse once at module load.

---

## 4. Low Priority Issues (Impact Score: 3-4)

### 4.1 Inconsistent error response shapes

Some endpoints return `{ error, message }` JSON, others return plain text `res.status(400).send("message")`, some return empty `res.status(400).send()`. Inconsistent across billing, user, and settings endpoints.

### 4.2 `checkBucksExtension` in `helper.js` makes HTTP call to own server

```js
httpProvider("GET", '/api/v1/public/themeAppEmbeds', fetch)
```
This is an internal round-trip (adds latency + serialization overhead). Instead, inline the logic.

### 4.3 `crisp-sdk-web` imported eagerly in `closePopover.jsx`

Rarely used (click feedback handler) but imported at the top level. Dynamic import would be cleaner.

### 4.4 `canvas-confetti` eagerly imported in `PricingCard.jsx`

Only needed when user applies a coupon. Dynamic import saves ~8KB.

### 4.5 `react-iframe` imported in preview component

`react-iframe` (~15KB) is a heavy wrapper for a native `<iframe>`. The component also has an unnecessary re-render issue (see below).

### 4.6 `preview-iframe` — `useEffect` triggers on unrelated store updates

`[userSettings, userDataGlobal]` dependencies — `userDataGlobal` is a Zustand store that changes reference on any unrelated update, causing iframe `postMessage` to fire unnecessarily.

**Fix**: Use a stable reference or compare specific fields.

### 4.7 `SelectCurrencies.jsx` — large array filtering on every keystroke

130-item currency list re-filtered via `RegExp` on every keystroke. Consider debouncing or memoizing.

### 4.8 `CollapsibleCard.jsx` — 18 `useState` hooks, complex lifecycle

Unusually high state count for a single component. Susceptible to stale closures.

### 4.9 Webhook routes missing HMAC validation

`pages/api/webhooks.js` and `[...webhookTopic].js` lack the `verifyHmac` middleware that the `v1/hooks/*` routes have.

---

## 5. Quick Wins (Low Effort, Measurable Impact)

| Rank | Task | Files | Est. Time | Impact |
|------|------|-------|-----------|--------|
| 1 | Fix `expires` field not persisted in `storeSession` | `utils/sessionHandler.js` | 5 min | Session expiry correctness |
| 2 | Add rate limiting to public endpoints | `middleware/rateLimit.js` + public routes | 30 min | Abuse prevention |
| 3 | Fix NoSQL injection in `dashboardSection.js` | `pages/api/v1/user/dashboardSection.js` | 5 min | Security |
| 4 | Add 60s TTL cache to widget hotpath endpoints | `config.js`, `appStatus.js` | 30 min | Reduces DB load 10-100x |
| 5 | Parallelize admin imports on 6 pages | `pages/settings`, `pricing`, `currency-rules`, `advanced`, `partners`, `index` | 20 min | Saves ~50ms per SSR |
| 6 | Fix pricing date inconsistency | `pages/settings/index.jsx` | 2 min | Correctness |
| 7 | Merge duplicate `users` queries in `getEffectiveShopContext` | `utils/admin/getEffectiveShopContext.js` | 10 min | Saves 1 DB round-trip |
| 8 | Remove debug `console.log(test)` | `utils/sessionHandler.js` | 1 min | Cleanup |
| 9 | Fix `filterDbData` object mutation | `utils/helper.js` | 5 min | Correctness |
| 10 | Convert Sentry imports to dynamic | `PartnerPromoBanner.jsx`, `AnalyticsBanner/index.jsx` | 15 min | Saves ~80KB bundle |
| 11 | Dynamically import TourGuide | `pages/analytics/index.jsx` | 5 min | Saves ~30KB bundle |
| 12 | Add timing to `admin/partners` pages | `admin/partners/index.jsx`, `[id].jsx` | 10 min | Debugging |
| 13 | Memoize env var parsing in `isSuperAdmin.js` | `utils/admin/isSuperAdmin.js` | 5 min | Micro-optimization |
| 14 | Fix `trialStartDat` typo | `pages/currency-rules/index.jsx` | 2 min | Correctness |

---

## 6. Architecture Recommendations (Medium-Long Term)

### 6.1 Implement request-scoped DB cache
Use `AsyncLocalStorage` + `Map<string, Promise>` to memoize `prisma.settings.findUnique({ where: { shop } })`, `prisma.users.findUnique(...)`, and `loadSessionWithShop(shop)` for the duration of a single SSR request. This eliminates ~4 duplicate DB queries per page load.

### 6.2 Adopt React Query for client-side data fetching
Replace manual `useFetch` + ad-hoc `useEffect` fetch patterns (especially on analytics page and widget hotpath API calls) with `react-query` — it's already installed. Gains: auto-caching, deduplication, retry, background refetch.

### 6.3 Add React.memo strategically
Zero components use `React.memo`. Adding it to chart components, `PricingCard`, `StatsCards`, and `PerformanceTables` would reduce unnecessary re-renders across navigation.

### 6.4 Consolidate partner banner queries
`pages/index.jsx` makes 3 separate `getActivePartners` DB queries for different categories. Accept an array of categories to combine into 1 query.

### 6.5 Standardize API error response format
All API endpoints should return `{ error: boolean, message: string }` JSON consistently. Currently billing endpoints use text, user endpoints use empty 400s, public endpoints use JSON — a mix of 3 patterns.

### 6.6 Add HMAC verification to legacy webhook routes
`pages/api/webhooks.js` and `[...webhookTopic].js` are missing `verifyHmac` middleware present in `v1/hooks/*`.
