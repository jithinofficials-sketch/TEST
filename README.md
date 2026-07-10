# Admin CLS Stabilization Design

## Context

Shopify admin-side CLS is measured for the embedded app experience in Shopify App Home. Shopify's Built for Shopify performance target for admin CLS is `0.1` or less at the 75th percentile.

The current admin dashboard assembles much of its visible layout after hydration through `next/dynamic()` imports, client-only responsive state, post-load banner state updates, and effect-driven onboarding checks. These patterns can cause layout elements to appear, collapse, expand, or reflow after the first paint.

This design targets the known priority CLS risks only. It does not change storefront widget behavior.

## Goals

- Reduce admin-side CLS on the embedded app dashboard.
- Keep the current dashboard UI and merchant-facing behavior intact.
- Make the initial server/client layout closer to the final hydrated layout.
- Avoid broad dashboard refactors while fixing the highest-risk layout shifts.
- Keep fixes compatible with the existing Polaris React UI in this Next.js app.

## Non-Goals

- Rebuilding the dashboard with Polaris web components.
- Changing storefront theme app extension or currency converter widget behavior.
- Redesigning dashboard content, pricing, analytics, or onboarding flows.
- Removing dynamic imports where stable fallbacks are sufficient.
- Adding a full analytics/performance telemetry system in this implementation.

## Scope

The implementation will address these priority items:

1. Add layout-preserving loading fallbacks to dashboard dynamic imports in `pages/index.jsx`.
2. Remove the client-only mobile layout flip in `pages/index.jsx`.
3. Make analytics banner rendering stable on first load.
4. Prevent `StaffAccessBar` from pushing content down after hydration.
5. Prevent onboarding validation from changing the card's page footprint immediately after first paint.
6. Fix invalid height styles in `components/home/feedbackCard.jsx` and `components/home/whatsNew.jsx`.

## Current Risks

### Dashboard Dynamic Imports

`pages/index.jsx` dynamically imports large dashboard sections with no loading fallback. These components render inside a Polaris `Grid`, often as full-width `Grid.Cell` children. When chunks load after the first paint, the page inserts new cards and banners into the layout.

Affected dynamic components include:

- `AnalyticsBanner`
- `PartnerPromoBanner`
- `OnboardingBanner`
- `FooterComponent`
- `AppStatus`
- `CollaborationBanner`
- `DashboardImageBanner`
- `FeedbackCard`
- `NeedAssistance`
- `HelpSupportCards`
- `OnboardingCompletionPopup`
- `WhatsNew`

### Client-Only Mobile Layout

`pages/index.jsx` initializes `isMobile` as `false`, then updates it from `window.innerWidth` in `useEffect`. On mobile, the CurrencyInfo/ExtensionCard area can first render as a row and then flip to a column after hydration.

### Analytics Banner Late Insertion

`pages/index.jsx` can set `analyticsWelcomeBanner` after first load through an API call and global state mutation. That can insert the analytics banner after the page has already painted.

### Staff Access Banner Late Insertion

`components/admin/StaffAccessBar.jsx` initially renders `null`, then checks impersonation state in `useEffect`. On impersonation routes, it can mount a banner above the page after hydration and push all content down.

### Onboarding Post-Load Reflow

`components/home/onBoardingBanner.jsx` delays validation checks by 100ms. Those checks update onboarding state, and `components/common/onBoardingCard.jsx` may collapse, expand, or hide large sections based on the result.

### Invalid Height Styles

Two components contain invalid CSS values, preventing intended layout reservation:

- `components/home/feedbackCard.jsx` has `height: '120px ,paddingBottom:16px'`.
- `components/home/whatsNew.jsx` has `minHeight: mdDown ? '204' : '230px'`.

## Design

### 1. Dashboard Dynamic Import Fallbacks

Create small dashboard fallback components in `pages/index.jsx` or a nearby component file. Each fallback will preserve the same grid footprint as the final component.

Fallback requirements:

- Use the same `Grid.Cell` `columnSpan` as the final component when the final component owns the cell.
- Use a Polaris `Card`, `Box`, or neutral block with stable `minHeight`.
- Avoid spinners as the only placeholder because spinners do not reserve final layout height.
- Keep placeholders visually lightweight and non-interactive.

For dynamic components rendered inside an existing `Grid.Cell`, the fallback should reserve only the child component's expected height. For dynamic components that render their own `Grid.Cell`, the fallback should render a matching `Grid.Cell`.

Expected fallback sizes:

- Partner promo banner: full-width cell, approximately `120px` minimum height.
- Onboarding banner: full-width cell, approximately `360px` minimum height when setup is incomplete.
- Analytics banner: full-width cell, approximately `140px` minimum height.
- App status: full-width cell, approximately `220px` minimum height.
- Need assistance: full-width cell, approximately `96px` minimum height.
- Collaboration banner: full-width cell, approximately `120px` minimum height.
- WhatsNew: full-width cell, approximately `230px` minimum height.
- Help support cards: full-width cell, approximately `96px` minimum height.
- Dashboard image banner: full-width cell, approximately `150px` minimum height.
- Feedback card: full-width cell, approximately `120px` minimum height.
- Footer: preserve a small footer block only if its late mount is visible in testing.

`OnboardingCompletionPopup` does not need a layout placeholder because it renders modal UI rather than occupying normal document flow.

### 2. CSS-Based Responsive Currency Cards

Replace the `isMobile` state used for the CurrencyInfo/ExtensionCard row with CSS.

Current pattern:

- Initial render assumes desktop row.
- `useEffect` checks `window.innerWidth`.
- State update changes `flexDirection` to column on mobile.

New pattern:

- Render a stable wrapper class, for example `bucks-dashboard-status-row`.
- Define row layout by default and column layout in a media query.
- Remove the resize listener and `isMobile` state.

This makes the first render and hydrated render structurally identical.

### 3. Stable Analytics Banner Rendering

Avoid inserting the analytics banner after first paint as a result of the first-load API mutation.

The first implementation should use this rule:

- The dashboard decides whether to render the analytics banner from initial server data for the current page load.
- If an API call sets `analyticsWelcomeBanner` for future eligibility, it must not insert the banner during the same initial render session.

This keeps current behavior predictable and avoids pushing dashboard content down. If product requirements require immediate display for newly eligible users, reserve the analytics banner slot from the start when onboarding is completed and the banner is not dismissed.

### 4. Stable Staff Access Banner Footprint

Prevent impersonation routes from gaining a new top banner after hydration without reserved space.

Use a route-based placeholder strategy:

- Detect whether the current route is an impersonation route from `router.pathname` or `router.asPath`.
- On possible impersonation routes, render a reserved wrapper with a stable minimum height.
- If impersonation data exists, render the banner inside the wrapper.
- If no impersonation data exists, the wrapper can collapse only for routes that are not impersonation routes.

This avoids shifting all page content when `StaffAccessBar` discovers impersonation state in `useEffect`.

### 5. Stable Onboarding Card Footprint

Keep the onboarding card footprint stable during post-load validation.

Rules:

- Use the server-provided onboarding state as the initial rendered state.
- Remove or reduce the visual impact of the 100ms delayed validation by keeping a stable wrapper `minHeight` while checks run.
- Do not collapse or remove the onboarding area immediately after first paint because a post-load validation result says all steps are complete.
- If all steps are complete in server data, render no onboarding area from the start.
- If server data says onboarding is incomplete, reserve the onboarding area while validation updates internal step status.

This preserves the existing validation behavior while preventing large top-of-page height changes.

### 6. Invalid Height Fixes

Fix malformed inline styles:

- In `components/home/feedbackCard.jsx`, replace the invalid `height` string with valid properties, such as `minHeight: '120px'` and `paddingBottom: '16px'`.
- In `components/home/whatsNew.jsx`, replace `'204'` with `'204px'`.

These changes are safe and directly improve layout reservation.

## Data Flow

### Dashboard Initial Render

1. `getServerSideProps` provides initial user/settings/banner state.
2. `pages/index.jsx` renders a stable dashboard grid from that state.
3. Dynamic sections that are not ready render layout-preserving fallbacks.
4. Hydration replaces fallbacks with real content of similar height.
5. Client-only validations update content inside reserved areas instead of adding or removing major layout blocks.

### Analytics Banner

1. Initial data determines current-page visibility.
2. First-load API mutation may update backend/global state for future sessions.
3. Current-page layout does not gain a new banner unless space was already reserved.

### Staff Access Banner

1. Route context determines whether a reserved banner area is needed.
2. Client-side impersonation lookup fills the reserved area if impersonation exists.
3. Non-impersonation routes keep current no-banner behavior.

## Error Handling

- Dynamic import fallbacks should not introduce new network or API behavior.
- Analytics banner API failures should preserve current error handling but must not trigger additional layout insertion.
- Onboarding validation failures should keep the server-derived onboarding UI and avoid collapse/removal based on failed checks.
- Staff impersonation lookup failures should leave either the reserved route-specific placeholder or no banner, depending on route type.

## Testing Plan

Manual verification:

- Run production build locally if environment allows: `npm run build` then `npm start`.
- Test embedded admin dashboard at desktop and mobile iframe widths.
- Use Chrome Performance panel with Layout Shift Regions enabled.
- Confirm dashboard does not shift when dynamic sections load.
- Confirm mobile dashboard does not flip CurrencyInfo/ExtensionCard from row to column after hydration.
- Confirm analytics banner does not appear after a first-load API mutation unless space was reserved.
- Confirm impersonation routes do not push content down after `StaffAccessBar` hydrates.
- Confirm onboarding validation updates step status without a large page-height change.

Code checks:

- Run the existing test suite if available: `npm test`.
- Run a production build: `npm run build`.

Optional measurement:

- Use Shopify App Bridge Web Vitals API or existing `web-vitals` logging to compare CLS before and after changes.

## Rollout Plan

1. Implement dashboard fallback and responsive layout fixes first.
2. Implement analytics banner stability.
3. Implement staff banner reservation.
4. Implement onboarding footprint stabilization.
5. Fix invalid height styles.
6. Build and manually verify layout shifts.

## Risks

- Reserved placeholders can leave small temporary blank areas if component chunks load slowly.
- Approximate fallback heights may need tuning after visual testing.
- Analytics banner behavior may change slightly for users who become eligible during the first page load; the banner may appear on the next load instead of immediately unless a reserved slot is used.
- Staff banner reservation must avoid adding blank vertical space to normal merchant routes.
- Onboarding stabilization must avoid hiding real completion progress from merchants.

## Open Decisions

- Use inline fallback components in `pages/index.jsx` for minimal change, unless the file becomes too noisy.
- Prefer reserving analytics banner space only for eligible users from SSR state, not for every dashboard user.
- Prefer route-based staff banner reservation only on `/admin/merchants/[merchant]/*` routes.

## Acceptance Criteria

- Dashboard dynamic imports have layout-preserving fallbacks.
- Dashboard CurrencyInfo/ExtensionCard row uses CSS responsive layout, not client-only `isMobile` state.
- Analytics banner cannot be newly inserted after first paint without reserved space.
- StaffAccessBar cannot push page content down on impersonation routes after hydration.
- Onboarding validation cannot collapse/remove a large onboarding area immediately after first paint.
- Invalid height styles in `feedbackCard.jsx` and `whatsNew.jsx` are corrected.
- `npm run build` completes successfully, or any build failure is documented as unrelated.
