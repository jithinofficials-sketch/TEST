# Auth Session Hardening Notes

## Problem

Normal merchant SSR pages currently use the URL query shop as the tenant key.

Example shape:

```js
const shop = context.query.shop;
const data = await loadShopData(shop);
```

Then SSR serializes merchant data to the browser:

```js
return {
  props: {
    initialData: {
      userData,
      settings,
    },
  },
};
```

`context.query.shop` is browser-controlled. If someone changes the URL to another installed shop, SSR can load data for that shop when the app has a stored offline session or DB records for it.

The stored offline session proves only this:

```text
The app has an installed token for that shop.
```

It does not prove this:

```text
The current browser requester belongs to that shop.
```

That is the merchant SSR security issue.

## Why The Strict SSR Fix Broke Install

The first attempted hardening was to use Shopify's request-bound session during merchant SSR:

```js
const session = await fetchSession({ req, res });
const shop = session.shop;
```

This is the right trust source because `session.shop` comes from the authenticated Shopify request, not from the URL.

But the normal embedded install flow lands on the dashboard through a browser redirect:

```text
/api/auth/callback
-> GET /?shop=store.myshopify.com&host=...
```

That first page request is not an App Bridge authenticated API request. It does not include:

```text
Authorization: Bearer <Shopify session token>
```

So `fetchSession({ req, res })` fails during SSR with an error like:

```text
Missing Authorization header, was the request made with authenticatedFetch?
```

Then the SSR page treats the fresh OAuth landing as unauthenticated and redirects to auth again. That creates an OAuth loop:

```text
OAuth completes
-> callback redirects to dashboard
-> dashboard SSR cannot fetch session
-> dashboard redirects to auth/exitframe
-> OAuth starts again
```

The problem is not that the OAuth callback failed. The callback completed and stored the session. The problem is that the next browser document request does not carry the App Bridge Authorization header that `fetchSession()` expects.

## Related Exitframe Bug

`pages/exitframe/[...shop].js` used to build auth URLs only from the Shopify browser config:

```js
const shop = window?.shopify?.config?.shop;

open(`${process.env.CONFIG_SHOPIFY_APP_URL}/api/auth?shop=${shop}`, "_top");
```

During install or reauth, `window.shopify.config.shop` can be missing. That produced:

```text
/api/auth?shop=undefined
```

and Shopify returned:

```text
InvalidShopError: Received invalid shop argument
```

The minimal fix is to read the catch-all route value first.

For a plain shop route:

```text
/exitframe/store.myshopify.com
```

build:

```text
/api/auth?shop=store.myshopify.com
```

For an existing OAuth query route:

```text
/exitframe/shop=store.myshopify.com&host=abc&embedded=1
```

preserve the query:

```text
/api/auth?shop=store.myshopify.com&host=abc&embedded=1
```

This fix only corrects OAuth URL construction. It does not solve the merchant SSR security issue by itself.

## Fix Option 1: SSR Shell + Authenticated Bootstrap API

This is the clean long-term fix.

Normal merchant SSR pages should not load sensitive merchant data. They should render a safe shell only.

Flow:

```text
1. OAuth completes.
2. /api/auth/callback redirects to /?shop=store.myshopify.com&host=...
3. SSR receives a normal browser GET with no Authorization header.
4. SSR renders a safe shell only.
5. App Bridge loads in the browser.
6. Client calls /api/v1/bootstrap using authenticated fetch.
7. Bootstrap API calls fetchSession({ req, res }).
8. Bootstrap API uses session.shop as the tenant key.
9. Bootstrap API loads merchant data and returns it.
10. Page renders the real dashboard/settings content.
```

SSR example:

```js
export async function getServerSideProps(context) {
  return {
    props: {
      shop: context.query?.shop || null,
      host: context.query?.host || null,
      initialData: null,
      needsBootstrap: true,
    },
  };
}
```

The SSR shell must not include sensitive merchant data:

```text
userData
settings
customCurrencyRules
partner_referral
billing or subscription data
merchant banner state
analytics flags
access tokens
```

Bootstrap API example:

```js
const session = await fetchSession({ req, res });

if (!session?.shop) {
  return res.status(401).json({ success: false, error: "Unauthorized" });
}

const shop = session.shop;
const data = await loadShopData(shop, { includeSetupStatus: true });

return res.status(200).json({ success: true, data });
```

Client example:

```js
const fetch = useFetch();

useEffect(() => {
  async function loadBootstrap() {
    const response = await fetch("/api/v1/bootstrap");
    const json = await response.json();

    if (json.success) {
      setUserDataGlobal(json.data.userData);
      setUserSettingsGlobal(json.data.settings);
    }
  }

  loadBootstrap();
}, []);
```

Why this fixes install:

```text
The initial browser GET does not need an Authorization header because SSR does not call fetchSession() or load merchant data.
```

Why this fixes security:

```text
Sensitive merchant data is loaded only by an authenticated API route. The API uses session.shop, not context.query.shop.
```

If an attacker opens:

```text
/?shop=victim.myshopify.com
```

SSR returns only the safe shell. The bootstrap API still returns data for `session.shop`, not for `victim.myshopify.com`.

## Fix Option 3: Short-Lived Signed OAuth Landing Cookie

This is a smaller transitional fix if the app must keep SSR data loading for now.

After OAuth callback succeeds, the server sets a short-lived signed cookie for the authenticated shop:

```text
recent_oauth_shop=signed(store.myshopify.com, timestamp)
```

Then the callback redirects:

```text
/?shop=store.myshopify.com&host=...
```

Merchant SSR checks:

```text
1. Try fetchSession({ req, res }).
2. If fetchSession() succeeds, use session.shop.
3. If fetchSession() fails, verify the recent OAuth landing cookie.
4. If cookie is valid and matches context.query.shop, allow this shop.
5. Otherwise, redirect and load no data.
```

Callback shape:

```js
const cookie = createSignedOAuthLandingCookie(session.shop);
res.setHeader("Set-Cookie", cookie);
res.redirect(`/?shop=${shop}&host=${host}`);
```

SSR shape:

```js
const session = await tryFetchSession(context);

if (session?.shop) {
  return session.shop;
}

const shopFromCookie = verifySignedOAuthLandingCookie({
  req: context.req,
  requestedShop: context.query.shop,
});

if (shopFromCookie) {
  return shopFromCookie;
}

return null;
```

Required guards:

```text
Cookie is signed with SHOPIFY_API_SECRET.
Cookie shop must match context.query.shop.
Cookie expires quickly, for example 60-120 seconds.
Cookie is HttpOnly, Secure, SameSite=None.
Cookie fallback applies only to normal merchant pages.
Cookie fallback must not apply to internal admin pages.
```

Why this fixes install:

```text
The first post-OAuth browser GET has no Authorization header, but it has a server-set signed cookie from the successful OAuth callback.
```

Why this fixes security:

```text
Changing ?shop is not enough. The requester also needs a fresh valid signed cookie for that same shop.
```

This option is faster but more delicate than Option 1 because the security depends on correct cookie signing, expiry, matching, and scoping.

## Recommendation

Use Option 1 for the permanent fix:

```text
SSR shell + authenticated bootstrap API
```

It is cleaner because sensitive data never leaves SSR based on `?shop`.

Use Option 3 only as a transition if rewriting all merchant pages to bootstrap data is too large for the current release.

## What Still Should Be Hardened Independently

These fixes do not depend on the merchant SSR strategy:

```text
Admin partner pages should authorize with fetchSession().shop, not query shop.
Merchant panel should authorize with isImpersonator(fetchSession().shop), not query shop or offline session fallback.
Public storefront validateUser should remain read-only and must not call saveAppMetafields().
Exitframe should preserve route shop or OAuth query params when building /api/auth URLs.
```
