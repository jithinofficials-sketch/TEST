# RFC: Merchant Panel

**Metadata:**
- **Author:** [Your Name]
- **Status:** Draft
- **Reviewers:** CPO, Product Owner
- **Type:** `#feature-rfc`

---

## 1. Problem

**We have no way to see what a merchant sees.**

When a merchant reports a bug or asks for help, we have to:
1. Ask them to send screenshots
2. Guess their settings from the database
3. Manually query MongoDB to check their config

This is slow, error-prone, and wastes both our time and the merchant's time.

**Current workarounds:**
- SSH into server, run Prisma queries to inspect merchant data
- Ask merchant to screen-share or send screenshots
- Reproduce issues locally with test data (often doesn't match real merchant state)

**What we need:**
A way for super admins to click one button and instantly see the app exactly as a specific merchant sees it — their settings, their analytics, their currency rules — without asking them anything.

---

## 2. Why This Matters?

**Why now:**
- Support tickets are growing. Every "can you send a screenshot?" adds 24+ hours to resolution.
- We already have the `isSuperAdmin` system and the `admin/partners` page pattern — the foundation exists.
- Multiple pages have inconsistent SSR logic (some check session validity, some don't) — fixing this alongside the panel avoids doing the same work twice.

**What we gain:**
- **Faster support resolution** — see merchant's exact state in seconds, not hours
- **Easier debugging** — reproduce issues with real merchant data
- **Consistent codebase** — standardized SSR logic across all pages (optimization bonus)
- **Foundation for future admin tools** — audit logs, bulk actions, etc.

**If we DON'T do this:**
- Support stays slow — keep asking merchants for screenshots
- Bugs stay harder to reproduce — guessing merchant state from raw DB queries
- SSR inconsistency grows — new pages copy-paste different patterns

---

## 3. Proposed Solution

### Core Idea

Super admin clicks "Access" on a merchant in the panel. The app sets a cookie + DB record, then redirects to `/?shop=merchant.myshopify.com`. Every page and API route detects the impersonation and loads that merchant's data instead.

### Why This Approach?

- **URL-based shop switching** — the app already uses `?shop=` to identify merchants. We just need to let admins use it for any shop.
- **Cookie + DB validation** — cookie for speed (no DB call on every client-side request), DB for security (validates the session is real).
- **Reuses existing systems** — `isSuperAdmin`, `getAppDetails`, `getUserDetails`, `fetchSession` all stay the same.

### New Systems Introduced

| System | Purpose |
|---|---|
| `withShopContext` wrapper | Standardizes all page SSR logic + admin bypass |
| `withImpersonation` middleware | Standardizes all API route auth + admin bypass |
| `getAdminViewContext` | Reads impersonation cookie, validates against DB |
| `impersonation_sessions` model | Tracks active impersonation sessions |
| `StaffAccessBar` component | Shows "Viewing as [shop]" banner + Exit button |

### What This Does NOT Do

- Does NOT give admin write access to Shopify APIs (e.g., can't create orders for merchants)
- Does NOT bypass Shopify's own authentication — only bypasses our internal session/availability checks
- Does NOT store merchant credentials — uses existing stored sessions

---

## 4. End-to-End Flow

### Admin Flow

```
Super Admin opens app
    |
    v
Sees normal dashboard (their own data)
    |
    v
Clicks "Merchants" in navigation
    |
    v
/admin/merchants page loads
Shows table: Shop | Name | Email | Plan | Status | [Access]
    |
    v
Clicks "Access" on a merchant
    |
    v
POST /api/admin/merchants/access
  - Validates admin is super admin
  - Creates impersonation_session in DB
  - Sets bucks_admin_imp cookie
    |
    v
Redirects to /?shop=merchant.myshopify.com (same tab)
    |
    v
Page loads with StaffAccessBar: "Viewing as merchant.myshopify.com [Exit]"
Admin sees merchant's dashboard, settings, analytics — everything
    |
    v
Clicks "Exit"
    |
    v
POST /api/admin/merchants/exit
  - Ends impersonation_session in DB
  - Clears cookie
    |
    v
Redirects back to /admin/merchants?shop=admin.myshopify.com
```

### Technical Flow — Page Load (SSR)

```
Browser requests /?shop=merchant.myshopify.com
    |
    v
getServerSideProps calls withShopContext(context)
    |
    v
withShopContext calls getAdminViewContext(context)
    |
    +--> Reads bucks_admin_imp cookie
    +--> Parses { adminShop, sessionId }
    +--> Validates adminShop is super admin
    +--> Checks impersonation_sessions DB: session active + not expired?
    +--> Confirms impersonatedShop matches URL's ?shop=
    |
    v
If valid admin view:
    - SKIP isShopAvailable (merchant might be uninstalled)
    - SKIP isSessionValid (merchant's token might be expired)
    - SKIP pricing redirect (admin doesn't need a plan)
    - Load merchant's data from DB directly
    - Return { adminCtx: { isAdminView: true, adminShop, ... } }
    |
    v
Page renders with merchant's data + StaffAccessBar
```

### Technical Flow — Client-Side API Call

```
Admin clicks "Save" on merchant's settings page
    |
    v
httpProvider reads bucks_admin_imp cookie from browser
Compares cookie's adminShop vs URL's ?shop=
If different → injects headers:
    X-Admin-Shop: admin.myshopify.com
    X-Impersonated-Shop: merchant.myshopify.com
    |
    v
POST /api/v1/settings (with impersonation headers)
    |
    v
withImpersonation middleware:
    1. Loads admin's session via fetchSession
    2. Reads X-Admin-Shop + X-Impersonated-Shop headers
    3. Validates admin is super admin
    4. Validates session.shop matches X-Admin-Shop
    5. Overrides req.effectiveShop = merchant.myshopify.com
    6. Loads merchant's accessToken for Shopify API calls
    |
    v
Handler uses req.effectiveShop → saves to merchant's settings
```

---

## 5. Implementation Details

### 5.1 Data Model

**New Prisma model: `impersonation_sessions`**

```prisma
model impersonation_sessions {
  id               String   @id @default(auto()) @map("_id") @db.ObjectId
  adminShop        String
  impersonatedShop String
  sessionId        String   @unique @default(uuid())
  isActive         Boolean  @default(true)
  createdAt        DateTime @default(now())
  expiresAt        DateTime
}
```

- Sessions expire after 24 hours
- Only one active session per admin at a time
- Old sessions are deactivated, not deleted (audit trail)

### 5.2 Backend — SSR Wrapper

**`utils/middleware/withShopContext.js`**

Replaces inline `getServerSideProps` logic in all 8 merchant pages.

| Option | Default | Purpose |
|---|---|---|
| `requireShop` | `true` | Run `isShopAvailable` check |
| `requireSession` | `true` | Run `isSessionValid` check |
| `pricingRedirect` | `true` | Redirect to /pricing if no plan |
| `fetchSettings` | `true` | Call `getAppDetails(shop)` |

Returns: `{ shop, userData, settings, superAdmin, adminCtx, redirect }`

**Optimization**: Pages that previously had NO session/shop checks (`settings`, `advanced`, `currency-rules`, `partners`, `pricing`) now get consistent checks via the wrapper. Admin bypass ensures this doesn't break impersonation.

### 5.3 Backend — API Middleware

**`utils/middleware/withImpersonation.js`**

Wraps all 12 merchant-facing API routes.

What it does:
1. Calls `fetchSession` to get admin's real session
2. Checks for `X-Admin-Shop` + `X-Impersonated-Shop` headers
3. If present: validates admin is super admin, overrides `req.effectiveShop`
4. If absent: normal flow, `req.effectiveShop = session.shop`

Each API changes two lines:
```js
// Before:
const session = await fetchSession({ req, res });
const { shop, accessToken } = session;

// After:
const { effectiveShop: shop, effectiveAccessToken: accessToken } = req;
```

**APIs to wrap:**
- `v1/settings/index.js`
- `v1/settings/status/updateStatus.js`
- `v1/user/onboarding.js`
- `v1/user/updateBannerStatus.js`
- `v1/user/skipOnboarding.js`
- `v1/user/latest.js`
- `v1/user/dashboardSection.js`
- `v1/user/updateChatToken.js`
- `v1/billing/initSubscription.js`
- `v1/billing/cancelSubscription.js`
- `v1/analytics/proxy.js`
- `v1/banner/dismiss.js`

### 5.4 Backend — Merchant Panel APIs

| Route | Method | Purpose |
|---|---|---|
| `/api/admin/merchants` | GET | List all merchants (name, email, plan, status) |
| `/api/admin/merchants/access` | POST | Start impersonation (create DB session + set cookie) |
| `/api/admin/merchants/exit` | POST | End impersonation (deactivate DB session + clear cookie) |

All three validate `isSuperAdmin` before proceeding.

### 5.5 Frontend — Client-Side Header Injection

**`utils/common/http/httpProvider.js`**

Add `getImpersonationHeaders()` function:
- Reads `bucks_admin_imp` cookie from `document.cookie`
- Gets target shop from URL `?shop=`
- If `adminShop !== targetShop` → returns `{ X-Admin-Shop, X-Impersonated-Shop }`
- Injected into every GET/POST/PUT/DELETE call

### 5.6 Frontend — Merchant List Page

**`pages/admin/merchants/index.jsx`**

- Gated by `getAuthorizedAdminShop` (same pattern as `admin/partners`)
- Table columns: Shop Domain | Store Name | Email | Plan | Status | Action
- Search/filter by shop domain or email
- "Access" button per row → calls `/api/admin/merchants/access` → redirects to `/?shop=target`

### 5.7 Frontend — StaffAccessBar

**`components/common/StaffAccessBar.jsx`**

- Fixed banner at top of every page during impersonation
- Shows: "Viewing as merchant.myshopify.com"
- "Exit" button → calls `/api/admin/merchants/exit` → redirects to `/admin/merchants?shop=adminShop`
- Only renders when `adminCtx.isAdminView === true`

### 5.8 Frontend — Navigation Update

**`components/common/Navigation.jsx`**

- Add "Merchants" tab after "Partners Dashboard" (super admin only)
- Links to `/admin/merchants?shop=${shop}`

### 5.9 Edge Cases

| Case | Handling |
|---|---|
| **Merchant uninstalled** | Admin bypass skips `isShopAvailable` — DB data still visible. Shopify API calls return errors → show "API unavailable" instead of crashing |
| **Merchant session expired** | Admin bypass skips `isSessionValid` — DB data visible. Shopify API calls fail gracefully |
| **Impersonation session expired (24h)** | `getAdminViewContext` returns `isAdminView: false` → normal checks kick in → admin redirected |
| **Non-admin sends fake headers** | `withImpersonation` validates `isSuperAdmin(adminShop)` AND `session.shop === adminShop` — rejects with 403 |
| **Admin opens multiple tabs** | Cookie is shared across tabs. Each tab reads same impersonation state. Switching merchants updates the cookie, all tabs reflect new merchant on next navigation |
| **Admin impersonates then admin panel is accessed** | `/admin/merchants` uses `getAuthorizedAdminShop` (checks real session), not the impersonation context |
| **Missing `?shop=` in URL** | `withShopContext` returns redirect early |
| **Cookie tampered** | DB validation in `getAdminViewContext` catches invalid `sessionId` |

---

## 6. Open Questions

1. **Should we log impersonation actions?** (e.g., "Admin viewed merchant's settings", "Admin saved settings for merchant") — useful for audit, but adds complexity.
2. **Should admin be able to make billing changes** (initSubscription, cancelSubscription) while impersonating? Could be dangerous. Consider making these read-only during impersonation.
3. **Should we add a search/filter on the merchants page?** With 100+ merchants, scrolling through a table isn't ideal. Probably yes, but scope it.
4. **Session expiry: 24 hours enough?** Could make this configurable via env var.

---

## References

- Existing admin pattern: `pages/admin/partners/index.jsx`
- Super admin check: `utils/admin/isSuperAdmin.js`
- Session handler: `utils/sessionHandler.js`

---

## Implementation Plan

| Phase | Goal | Steps | Risk |
|---|---|---|---|
| **Phase 1: Wrapper** | Standardize SSR logic, no behavior change | Create `withShopContext.js`, update 8 pages | Medium — test each page |
| **Phase 2: API Middleware** | Standardize API auth, no behavior change | Create `withImpersonation.js`, update 12 APIs | Medium — test each API |
| **Phase 3: Impersonation Core** | DB + cookie + validation logic | Prisma model, CRUD helpers, `getAdminViewContext` | Low |
| **Phase 4: Merchant Panel UI** | Admin can list + access merchants | Merchant APIs, list page, StaffAccessBar, Navigation | Low |
| **Phase 5: Wire Everything** | Connect admin bypass to wrapper + middleware | Update `withShopContext` + `withImpersonation` to use `getAdminViewContext` | Low |
| **Phase 6: Testing** | Full flow verification | Normal merchant flow + admin impersonation flow | — |

Each phase ships independently. Phase 1-2 are pure refactors (zero behavior change). Phase 3-5 add the merchant panel feature.
