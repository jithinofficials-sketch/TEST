# Next.js 16.2 — What's New & How It Benefits Our Codebase

> **Presenter:** Software Developer  
> **Current version:** `next@^14.2.3` (Pages Router)  
> **Target version:** `next@16.2` (latest)  
> **React:** Currently `18.2.0` → needs `19.2`

---

## 1. Why Upgrade?

| Area | v14 (Current) | v16.2 |
|------|--------------|-------|
| Bundler | Webpack (only) | **Turbopack by default** (dev + build) |
| Dev startup | Baseline | **~400% faster** `next dev` startup |
| Server rendering | Baseline | **25–60% faster** HTML rendering (RSC payload deserialization) |
| React | 18.2 | **19.2** (View Transitions, React Compiler, Activity API) |
| Middleware | `middleware.js` | Renamed to **`proxy.js`** (Node.js runtime, no edge) |
| Caching | Route-level | Per-segment prefetch + `cacheComponents` |
| Dev/Build output | Shared `.next/` | **Separate** `.next/dev` + `.next/` (concurrent dev & build) |

---

## 2. Top Features Relevant to Our Codebase

### 2.1 Turbopack by Default
- No more `--turbopack` flag needed — it's the default for both `next dev` and `next build`.
- **⚠️ Impact on us:** We have a custom `webpack` config in `next.config.js` (lines 53-70) for `resolve.fallback` (fs, net, tls, readline) and externalizing `posthog-node`.
- **Migration path:**
  - Either use `--webpack` flag temporarily: `"build": "next build --webpack"`
  - Or migrate to Turbopack's `resolveAlias`:
    ```js
    // next.config.js — Turbopack equivalent
    const nextConfig = {
      turbopack: {
        resolveAlias: {
          fs: { browser: './empty.js' },
          net: { browser: './empty.js' },
          tls: { browser: './empty.js' },
          readline: { browser: './empty.js' },
        },
      },
    };
    ```
- **Turbopack FS Caching (beta):** Stores compiler artifacts on disk between restarts → even faster recompiles.
  ```js
  experimental: { turbopackFileSystemCacheForDev: true }
  ```

### 2.2 ~400% Faster Dev Startup
- `next dev` now loads config only once (was twice before).
- **Note:** `process.argv` no longer includes `'dev'` during config load — use `process.env.NODE_ENV === 'development'` instead.  
  ✅ Our `next.config.js` already uses `process.env.NODE_ENV` — no change needed.

### 2.3 25–60% Faster Server Rendering
- React's `JSON.parse` reviver callback was causing C++/JS boundary overhead in V8.
- New two-step approach: plain `JSON.parse()` + recursive JS walk.
- **Real-world results:**
  - Table with 1000 items → **26% faster**
  - Nested Suspense → **33% faster**
  - Rich text pages → **60% faster**
- **Impact:** All our API routes and SSR pages benefit automatically — zero code changes.

### 2.4 Middleware → Proxy Rename
- `middleware.js` is **deprecated**, renamed to `proxy.js`.
- Exported function `middleware()` → `proxy()`.
- Config: `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`.
- **Our middleware.js** (33 lines, CSP + CORS headers) → simple rename.
  ```
  mv middleware.js proxy.js
  # Rename function: middleware() → proxy()
  ```
- **Runtime change:** `proxy.js` runs on Node.js only (not Edge). Our middleware doesn't use Edge-specific APIs, so this is fine.

### 2.5 React 19.2 + React Compiler
- **React Compiler (stable):** Automatic memoization — no more manual `useMemo`/`useCallback` needed.
  ```js
  // next.config.js
  const nextConfig = { reactCompiler: true };
  ```
  - Potential impact: We use `react-query` with various callbacks — compiler auto-memoizes them.
  
- **View Transitions API:** Native page transition animations.
  ```jsx
  <Link href="/settings" transitionTypes={['slide']}>Settings</Link>
  ```
  - Could enhance our Navigation component for smooth page transitions.

- **`useEffectEvent`:** Extract non-reactive logic from Effects — useful for our PostHog tracking in `_app.js`.

- **Activity API:** Hide UI with `display: none` while maintaining state — useful for tab-based interfaces.

### 2.6 Enhanced Routing & Navigation
- **Layout deduplication:** Shared layouts downloaded once across sibling routes.
- **Incremental prefetching:** Only prefetches parts not already in cache.
- **Impact:** Our dashboard pages share a common layout — fewer network requests, faster navigations.

### 2.7 Better DX (Developer Experience)
| Feature | Benefit |
|---------|---------|
| **Server Function Logging** | See function name, args, execution time in terminal |
| **Hydration Diff Indicator** | `+ Client / - Server` legend in error overlay |
| **Error Causes in Dev Overlay** | `Error.cause` chains shown up to 5 levels deep |
| **`--inspect` for `next start`** | Debug/profile production server with Node.js debugger |
| **New Default Error Page** | Cleaner 500 page out of the box |

### 2.8 Experimental Features Worth Watching
- **`unstable_catchError()`** — Granular component-level error boundaries (better than just `error.js`).
- **`unstable_retry()`** — Built-in retry logic for error recovery (re-fetches data, not just re-renders).
- **`experimental.prefetchInlining`** — Bundle all segment data into single response per link.
- **`experimental.appNewScrollHandler`** — Better scroll/focus management after navigation.

---

## 3. Breaking Changes We Need to Handle

### 3.1 Async Request APIs (Fully Enforced)
In v15 it was a warning, in v16 synchronous access is **removed**:
- `cookies()`, `headers()`, `draftMode()` — must be `await`ed
- `params` and `searchParams` in pages/layouts — must be `await`ed

```js
// Before (v14)
export default function Page({ searchParams }) {
  const shop = searchParams.shop;
}

// After (v16)
export default async function Page({ searchParams }) {
  const { shop } = await searchParams;
}
```
**Impact:** All our `pages/` directory API routes and pages that access these need updating. The codemod handles this automatically:
```bash
npx @next/codemod@canary upgrade latest
```

### 3.2 React 18 → 19 Upgrade
- `react` and `react-dom` must be upgraded from `18.2.0` to `19.2`.
- `react-query v3` (we use `^3.39.3`) — **needs migration to `@tanstack/react-query` v5** for React 19 compatibility.

### 3.3 Node.js Minimum Version
- Requires **Node.js 20.9.0+**

### 3.4 Removed Features
| Removed | Impact on Us |
|---------|-------------|
| AMP support | None — we don't use AMP |
| `next lint` command | Use ESLint/Biome directly; `next build` no longer runs lint |
| `next/legacy/image` | None — we use `next/image` |
| Runtime config (`publicRuntimeConfig`) | Check if we use it |

### 3.5 Custom Webpack Config
Our `next.config.js` has a `webpack` callback. In v16, `next build` uses Turbopack by default and will **fail** if it finds a webpack config.
- **Option A:** Add `--webpack` flag to build script
- **Option B:** Migrate to Turbopack config (recommended long-term)

---

## 4. Migration Effort Estimate

| Task | Effort | Priority |
|------|--------|----------|
| Upgrade `next`, `react`, `react-dom` | Low | High |
| Run async API codemod | Low | High |
| Rename `middleware.js` → `proxy.js` | Low | High |
| Migrate `react-query` v3 → `@tanstack/react-query` v5 | **Medium** | High |
| Migrate webpack config → Turbopack | Low-Medium | Medium |
| Test all pages/API routes | Medium | High |
| Enable React Compiler | Low | Low (optional) |
| Add View Transitions | Low | Low (nice-to-have) |

### Recommended Migration Steps
1. **Backup & branch** — create `feature/next16-upgrade`
2. **Run the codemod:**
   ```bash
   npx @next/codemod@canary upgrade latest
   ```
3. **Update dependencies manually:**
   ```bash
   npm install next@latest react@latest react-dom@latest
   npm install @tanstack/react-query@latest  # replace react-query
   ```
4. **Rename middleware → proxy**
5. **Handle webpack → Turbopack** (or use `--webpack` flag)
6. **Test thoroughly** — all API routes, pages, widget loading
7. **Enable optional features** — React Compiler, View Transitions

---

## 5. Quick Wins After Upgrade (Zero/Low Effort, High Impact)

1. **Free performance** — 25-60% faster rendering, 400% faster dev startup
2. **Turbopack FS caching** — one config line for faster rebuilds
3. **React Compiler** — auto-memoization, one config line
4. **Better debugging** — Server Function logging, Hydration diffs, Error causes
5. **`--inspect` on production** — debug production issues easily

---

## 6. References

- [Next.js 16.2 Blog Post](https://nextjs.org/blog/next-16-2)
- [Upgrading to v16 Guide](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [Next.js Blog (all releases)](https://nextjs.org/blog)
- [React 19.2 Announcement](https://react.dev/blog/2025/10/01/react-19-2)
- [Turbopack Deep Dive](https://nextjs.org/blog/next-16-2-turbopack)

---

> **TL;DR:** Upgrading from Next.js 14 → 16.2 gives us **massive free performance gains** (400% faster dev, 25-60% faster rendering), **Turbopack** as the default bundler, **React 19.2** with auto-memoization via React Compiler, better DX with debugging tools, and a cleaner architecture (`proxy.js` replacing middleware). The biggest migration effort is upgrading `react-query` v3 → `@tanstack/react-query` v5 and handling the async Request APIs codemod.

