# Authentication Process Documentation

## Overview

This document explains how authentication works in our Shopify app. The authentication process uses **Shopify OAuth 2.0** to securely connect merchants' stores to our app.

---

## Table of Contents

1. [High-Level Flow](#high-level-flow)
2. [Frontend Authentication](#frontend-authentication)
3. [Backend Authentication](#backend-authentication)
4. [Token Management](#token-management)
5. [Session Validation](#session-validation)
6. [Security Features](#security-features)

---

## High-Level Flow

When a merchant installs or uses our app, here's what happens:

1. **Merchant clicks "Install App"** → Redirected to our app
2. **App initiates OAuth** → Shopify asks merchant to approve permissions
3. **Merchant approves** → Shopify sends authorization code back
4. **App exchanges code for tokens** → Gets access token from Shopify
5. **App stores session** → Saves session data in database
6. **App validates requests** → Checks session on every API call

---

## Frontend Authentication

### Step 1: Initial App Load

**File:** `pages/_app.js`

When the app loads in the Shopify admin:
- The app is embedded in an iframe inside Shopify admin
- Polaris Provider wraps the entire app for UI components
- No explicit login form - authentication happens automatically via OAuth

### Step 2: Exit Frame (Re-authentication)

**File:** `pages/exitframe/[...shop].js`

**When it happens:** When session expires or is invalid

**What it does:**
1. Breaks out of the Shopify iframe
2. Redirects to `/api/auth?shop={shop}` in the top window
3. Shows a loading screen: "Security Checkpoint - Reauthorizing your tokens"

**Why:** Shopify requires OAuth to happen outside the iframe for security

---

## Backend Authentication

### Step 1: Start OAuth Flow

**File:** `pages/api/auth/index.js`

**Trigger:** Merchant visits `/api/auth?shop={shop-name}`

**What happens:**

1. **Check if shop parameter exists**
   - If missing → Return error "No shop provided"

2. **Handle embedded app scenario**
   - If `embedded=1` query param exists
   - Redirect to `/exitframe/` to break out of iframe
   - This ensures OAuth happens in top window

3. **Begin OAuth process**
   ```javascript
   shopify.auth.begin({
     shop: req.query.shop,
     callbackPath: `/api/auth/tokens`,
     isOnline: false,
     rawRequest: req,
     rawResponse: res,
   })
   ```
   - `isOnline: false` → Request offline access token (doesn't expire when merchant logs out)
   - Redirects merchant to Shopify's authorization page
   - Shopify will callback to `/api/auth/tokens`

**Error handling:**
- `CookieNotFound` → Retry auth
- `InvalidOAuthError` → Retry auth
- `InvalidSession` → Retry auth
- `BotActivityDetected` → Block with 410 status

---

### Step 2: Token Exchange

**File:** `pages/api/auth/tokens.js`

**Trigger:** Shopify redirects here after merchant approves permissions

**What happens:**

1. **Exchange authorization code for access token**
   ```javascript
   const callbackResponse = await shopify.auth.callback({
     rawRequest: req,
     rawResponse: res,
   })
   ```
   - Shopify SDK handles the token exchange automatically
   - Returns a `session` object with access token

2. **Register webhooks**
   ```javascript
   await shopify.webhooks.register({ session })
   ```
   - Sets up webhooks for app uninstall, shop updates, etc.

3. **Start second OAuth flow**
   - Redirects to `/api/auth/callback`
   - This completes the full authentication cycle

---

### Step 3: Final Callback & Session Storage

**File:** `pages/api/auth/callback.js`

**Trigger:** After token exchange completes

**What happens:**

1. **Complete OAuth callback**
   ```javascript
   const callbackResponse = await shopify.auth.callback({
     rawRequest: req,
     rawResponse: res,
   })
   const { session } = callbackResponse
   ```

2. **Extract session data**
   - `shop` → Store domain (e.g., "example.myshopify.com")
   - `accessToken` → OAuth access token
   - `scope` → Permissions granted
   - `expires` → Session expiration date
   - `isOnline` → Whether it's an online or offline token

3. **Store session in database**
   ```javascript
   await sessionHandler.storeSession(session)
   ```
   - Saves to `shopify_sessions` table in database
   - Includes: uuid, shop, state, isOnline, scope, accessToken

4. **Check if user exists**
   - Query database for existing user by shop domain
   - If new user → Call `afterAuth()` to set up user account
   - If existing user → Update trial dates if applicable

5. **Redirect to app**
   ```javascript
   res.redirect(`/?shop=${shop}&host=${host}`)
   ```
   - Merchant is now authenticated and can use the app

---

### Step 4: After Authentication Setup

**File:** `pages/api/auth/afterAuth.js`

**Purpose:** Set up new user accounts and fetch shop data

**What happens:**

1. **Determine if new user**
   - Check if user has `is_active` flag in database
   - New users don't have this flag set

2. **For new users:**
   - Fetch order count from Shopify API
   - Store order history data
   - Set `last_installed` timestamp

3. **Fetch shop details**
   ```javascript
   const { shop: shopData } = await restResources(session, {
     path: 'shop',
   })
   ```
   - Gets shop name, email, currency, timezone, etc.

4. **Update trial dates**
   - Calculate trial start and end dates
   - Store in database

5. **Save to database**
   - Upsert user record with all shop data
   - Set `is_active: true`
   - Store access token

6. **Log installation**
   - Send notification to Slack about new installation

---

## Token Management

### Token Generation

**Who generates:** Shopify (not our app)

**When:** During OAuth flow after merchant approves permissions

**Types:**
- **Offline Access Token** (what we use)
  - Doesn't expire when merchant logs out
  - Used for background tasks
  - Stored in database
  
- **Online Access Token** (not used in this app)
  - Expires when merchant logs out
  - Used for user-specific actions

### Token Storage

**File:** `utils/sessionHandler.js`

**Database table:** `shopify_sessions`

**What's stored:**
```javascript
{
  uuid: session.id,           // Unique session ID
  shop: session.shop,         // Store domain
  state: session.state,       // OAuth state parameter
  isOnline: session.isOnline, // false for offline tokens
  scope: session.scope,       // Permissions granted
  accessToken: session.accessToken // The actual token
}
```

**Storage method:**
```javascript
const storeSession = async (session) => {
  await prisma.shopify_sessions.upsert({
    where: { uuid: session.id },
    update: { /* session data */ },
    create: { /* session data */ }
  })
}
```

**Security:** 
- Access tokens are stored in the database
- Database should be secured with proper access controls
- Consider encrypting tokens at rest (encryption code is commented out but available)

### Token Retrieval

**Function:** `loadSession(uuid)`

**How it works:**
1. Query database by session UUID
2. If found → Create Shopify Session object
3. If not found → Return undefined

**Usage:**
```javascript
const sessionId = await shopify.session.getCurrentId({
  isOnline: false,
  rawRequest: req,
  rawResponse: res,
})
const session = await sessionHandler.loadSession(sessionId)
```

### Token Expiry

**Offline tokens:** Don't expire automatically, but can be revoked

**Session expiry check:**
```javascript
const isExpired = new Date(session?.expires) < new Date()
```

**What happens when expired:**
- Session validation fails
- User is redirected to `/exitframe/` 
- OAuth flow starts again
- New token is generated

---

## Session Validation

### Middleware Protection

**File:** `middleware.js` (Next.js middleware)

**Purpose:** Set security headers for all pages

**What it does:**
1. Validates shop parameter format
2. Sets Content-Security-Policy header
   - Allows embedding in Shopify admin iframe
   - Blocks embedding in other sites
3. Runs on every page request (except auth/webhook routes)

**Protected routes:** All pages except:
- `/api/auth/*`
- `/api/webhooks/*`
- `/api/proxy_route`
- `/api/gdpr`
- Static files

---

### Request Verification Middleware

**File:** `utils/middleware/verifyRequest.js`

**Purpose:** Verify every API request has a valid session

**When it runs:** On protected API routes

**Validation steps:**

1. **Get current session ID**
   ```javascript
   const sessionId = await shopify.session.getCurrentId({
     isOnline: false,
     rawRequest: req,
     rawResponse: res,
   })
   ```

2. **Load session from database**
   ```javascript
   const session = await sessionHandler.loadSession(sessionId)
   ```

3. **Check session validity**
   ```javascript
   if (
     new Date(session?.expires) > new Date() &&
     shopify.config.scopes.equals(session.scope)
   ) {
     // Session is valid
   }
   ```
   - Checks if session hasn't expired
   - Checks if scopes haven't changed

4. **Test session with Shopify API**
   ```javascript
   const client = new shopify.clients.Graphql({ session })
   await client.query({ data: TEST_QUERY })
   ```
   - Makes a test GraphQL query to Shopify
   - If successful → Session is definitely valid
   - If fails → Session is invalid

5. **If valid:**
   - Attach session to request: `req.user_session = session`
   - Set CSP header for iframe security
   - Allow request to proceed

6. **If invalid:**
   - Check for Bearer token in Authorization header
   - Decode session token to get shop
   - Return 403 with reauth URL
   - Or redirect to `/exitframe/`

**Error handling:**
- Catches all errors
- Returns 401 Unauthorized
- Logs error details

---

### Page-Level Session Validation

**File:** `utils/middleware/isSessionValid.js`

**Purpose:** Check session validity in `getServerSideProps`

**When to use:** On pages that need authentication

**How it works:**

1. **Load session by shop**
   ```javascript
   const session = await sessionHandler.loadSessionWithShop(shop)
   ```

2. **Check if session exists**
   ```javascript
   if (!session) {
     return { redirect: { data: true } }
   }
   ```

3. **Check if expired**
   ```javascript
   const isExpired = new Date(session?.expires) < new Date()
   if (isExpired) {
     return { redirect: { data: true } }
   }
   ```

4. **Return validation result**
   - `{ redirect: { data: false } }` → Session valid
   - `{ redirect: { data: true } }` → Need to re-authenticate

---

### Client Provider (API Helpers)

**File:** `utils/clientProvider.js`

**Purpose:** Create authenticated API clients for making Shopify API calls

**Functions:**

1. **fetchSession** - Get current session
   ```javascript
   const session = await fetchSession({ req, res, isOnline: false })
   ```

2. **graphqlClient** - Create GraphQL client
   ```javascript
   const { client, shop, session } = await graphqlClient({ req, res, isOnline: false })
   // Use client to make GraphQL queries
   ```

3. **restClient** - Create REST client
   ```javascript
   const { client, shop, session } = await restClient({ req, res, isOnline: false })
   // Use client to make REST API calls
   ```

**Usage in API routes:**
```javascript
export default async function handler(req, res) {
  const { client, shop } = await graphqlClient({ req, res, isOnline: false })
  
  const response = await client.query({
    data: `{ products(first: 10) { edges { node { id title } } } }`
  })
  
  res.json(response.body)
}
```

---

## Security Features

### 1. OAuth 2.0 Flow
- Industry-standard authentication protocol
- Merchant never shares password with our app
- Shopify controls token generation

### 2. Offline Access Tokens
- Don't expire when merchant logs out
- Can be revoked by merchant at any time
- Stored securely in database

### 3. Scope Validation
- App requests specific permissions (scopes)
- If scopes change, re-authentication required
- Prevents privilege escalation

### 4. Session Expiry Checks
- Every request validates session hasn't expired
- Automatic re-authentication when needed

### 5. Content Security Policy (CSP)
- Prevents clickjacking attacks
- Only allows embedding in Shopify admin
- Set via middleware on every request

### 6. HMAC Verification
- Webhooks and proxy routes verify HMAC signatures
- Ensures requests actually come from Shopify
- Prevents request forgery

### 7. Frame Ancestors Control
```javascript
frame-ancestors 'self' https://admin.shopify.com https://${shop};
```
- Controls where app can be embedded
- Prevents malicious iframe embedding

### 8. Error Handling
- Graceful handling of auth errors
- Automatic retry for recoverable errors
- Bot detection and blocking

---

## Common Scenarios

### Scenario 1: First-Time Installation

1. Merchant clicks "Install App" in Shopify App Store
2. Redirected to `/api/auth?shop={shop}`
3. OAuth flow begins → Shopify asks for permissions
4. Merchant approves → Shopify redirects to `/api/auth/tokens`
5. App exchanges code for access token
6. Webhooks registered
7. Redirected to `/api/auth/callback`
8. Session stored in database
9. User account created via `afterAuth()`
10. Merchant redirected to app homepage

### Scenario 2: Returning User

1. Merchant opens app from Shopify admin
2. App loads in iframe
3. Middleware sets CSP headers
4. Page makes API request
5. `verifyRequest` middleware checks session
6. Session is valid → Request proceeds
7. Page renders with data

### Scenario 3: Expired Session

1. Merchant opens app (session expired)
2. Page makes API request
3. `verifyRequest` middleware checks session
4. Session expired → Returns 403 with reauth URL
5. Frontend redirects to `/exitframe/`
6. Breaks out of iframe
7. Redirects to `/api/auth?shop={shop}`
8. OAuth flow starts again
9. New session created
10. Merchant redirected back to app

### Scenario 4: Scope Change

1. App developer updates required scopes in config
2. Merchant opens app
3. `verifyRequest` checks if scopes match
4. Scopes don't match → Triggers re-authentication
5. OAuth flow requests new permissions
6. Merchant approves new scopes
7. New session with updated scopes stored

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `pages/api/auth/index.js` | Start OAuth flow |
| `pages/api/auth/tokens.js` | Exchange code for token |
| `pages/api/auth/callback.js` | Complete OAuth, store session |
| `pages/api/auth/afterAuth.js` | Set up new user accounts |
| `pages/exitframe/[...shop].js` | Break out of iframe for OAuth |
| `utils/sessionHandler.js` | Store/load sessions from database |
| `utils/middleware/verifyRequest.js` | Validate session on API requests |
| `utils/middleware/isSessionValid.js` | Validate session on pages |
| `utils/clientProvider.js` | Create authenticated API clients |
| `utils/shopify.js` | Shopify SDK configuration |
| `middleware.js` | Next.js middleware for security headers |

---

## Troubleshooting

### "No shop provided" error
- Shop parameter missing from URL
- Ensure installation URL includes `?shop={shop-name}`

### Infinite redirect loop
- Session not being stored properly
- Check database connection
- Verify session storage function

### "Session expired" constantly
- Check system clock is correct
- Verify session expiry dates in database
- Check if offline tokens are being used

### 403 Forbidden errors
- Session validation failing
- Check if scopes have changed
- Verify access token is valid
- Test with Shopify API directly

### App won't load in iframe
- CSP headers not set correctly
- Check middleware.js configuration
- Verify shop parameter format

---

## Best Practices

1. **Always use offline tokens** for background tasks
2. **Validate sessions on every API request** using middleware
3. **Handle re-authentication gracefully** with exitframe
4. **Store tokens securely** in database (consider encryption)
5. **Log authentication events** for debugging
6. **Test OAuth flow** after scope changes
7. **Monitor session expiry** and handle proactively
8. **Use client providers** instead of creating clients manually

---

## Summary

The authentication process in our app:

1. **Uses Shopify OAuth 2.0** for secure merchant authentication
2. **Stores offline access tokens** in database for persistent access
3. **Validates every request** using middleware
4. **Handles re-authentication** automatically when sessions expire
5. **Secures the app** with CSP headers and scope validation
6. **Provides helper functions** for creating authenticated API clients

The entire flow is designed to be **secure, automatic, and user-friendly** - merchants don't need to remember passwords or manually log in.
