# Dashboard Review Feedback Implementation Plan

> **For implementers:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task.

**Goal:** Replace the Bucks dashboard Good/Bad feedback card with the Figma star-review card, route five-star feedback through Shopify Reviews with an App Store fallback, and send lower ratings as silent structured Crisp feedback.

**Architecture:** Keep `FeedbackCard` as the single feature boundary at the existing dashboard placement. It will use `useAppBridge()` to access `shopify.reviews.request()`, manage review-request and feedback-modal state locally, and retain `ClosePopover` as the only persisted dismissal path. Add narrowly scoped Vitest component tests that mock App Bridge, Crisp, and PostHog.

**Tech Stack:** Next.js, React 18, Shopify Polaris 12, Shopify App Bridge React 4, Crisp SDK, PostHog browser client, Vitest, React Testing Library.

**Approved Figma references:**

- Dashboard review card: [node 15426:40884](https://www.figma.com/design/zWmT6pWcnApOQrtvHv3KC5/UFE-Widget---UFE-3.0--ALPHA-2.0?node-id=15426-40884&t=9oNOFO8taGndXOgN-4)
- Custom feedback modal: [node 15426:42089](https://www.figma.com/design/zWmT6pWcnApOQrtvHv3KC5/UFE-Widget---UFE-3.0--ALPHA-2.0?node-id=15426-42089&t=9oNOFO8taGndXOgN-4)

---

### Task 1: Add a focused feedback-card test harness

**Files:**
- Create: `tests/home/FeedbackCard.test.jsx`
- Reference: `tests/home/UserGuideCard.test.jsx`
- Reference: `tests/common/closePopover.test.jsx`
- Reference: `components/home/feedbackCard.jsx`

**Step 1: Write the shared mocks and Polaris render helper**

Create `tests/home/FeedbackCard.test.jsx`. Mock these boundaries before importing the component:

```jsx
const mocks = {
  reviewRequest: vi.fn(),
  triggerMessage: vi.fn(),
  closePopover: vi.fn(),
};

vi.mock("@shopify/app-bridge-react", () => ({
  useAppBridge: () => ({ reviews: { request: mocks.reviewRequest } }),
}));

vi.mock("@/utils/extras/crispChat", () => ({
  triggerMessage: mocks.triggerMessage,
}));

vi.mock("@/components/common/closePopover", () => ({
  default: ({ closeAction }) => (
    <button type="button" onClick={() => closeAction(false)}>Dismiss card</button>
  ),
}));
```

Render the component inside Polaris `AppProvider` with `enTranslations`, matching `tests/home/UserGuideCard.test.jsx`. Use a fixed merchant object:

```js
const user = { myshopify_domain: "test.myshopify.com" };
```

Stub `window.open` in `beforeEach`, clear all mocks, and remove `window.posthog` after every test.

**Step 2: Run the new test file to confirm the harness loads**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx
```

Expected: the command fails because the new component behavior and its test cases have not yet been added, or passes only the initial render smoke test if added first.

**Step 3: Commit the test harness**

```powershell
git add tests/home/FeedbackCard.test.jsx
```

### Task 2: Specify the failing lower-rating feedback tests

**Files:**
- Modify: `tests/home/FeedbackCard.test.jsx`
- Modify: `components/home/feedbackCard.jsx`
- Reference: `utils/extras/crispChat.js:282-291`
- Reference: `utils/common/constants/constants.js`

**Step 1: Write the failing modal-opening test**

Add a test that clicks the button labelled `Rate BUCKS 3 out of 5 stars` and verifies:

```jsx
expect(screen.getByRole("dialog", { name: "Share your feedback" })).toBeInTheDocument();
expect(screen.queryByText(/3 out of 5/i)).not.toBeInTheDocument();
expect(screen.getByLabelText("Hard to setup")).toBeInTheDocument();
expect(screen.getByLabelText("What can we improve?")).toBeInTheDocument();
```

Also assert the card title remains rendered behind the modal.

**Step 2: Write the failing silent-Crisp payload tests**

Add four tests for Send:

- reasons only: select `Hard to setup` and `Missing features`;
- text only: enter `The setup instructions are unclear.`;
- reasons and text;
- empty feedback.

Each test must assert `triggerMessage` is called with `REVIEW_QUERY`, `user`, and one exact formatted message. For example:

```js
expect(mocks.triggerMessage).toHaveBeenCalledWith(
  REVIEW_QUERY,
  user,
  "Dashboard feedback\nReasons: Hard to setup, Missing features\nWhat can we improve: The setup instructions are unclear."
);
```

Assert Send closes the dialog, leaves the card visible, and does not call `window.open`.

**Step 3: Write the failing cancel tests**

Add one test for Cancel and one for the modal close button. Both must assert that `triggerMessage` was not called and that the feedback dialog closes while the card remains visible.

**Step 4: Run the lower-rating tests to verify failure**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx
```

Expected: FAIL because the current Good/Bad card has no stars or custom feedback modal.

**Step 5: Implement the minimal lower-rating flow in `FeedbackCard`**

Replace the Good/Bad button group in `components/home/feedbackCard.jsx` with:

- Five `<Button variant="tertiary">` star buttons, using Polaris star icons and `accessibilityLabel={`Rate BUCKS ${rating} out of 5 stars`}`.
- Local state for `isFeedbackModalOpen`, selected feedback reasons, and feedback text.
- A shared `FEEDBACK_REASONS` array in this component in Figma order:

```js
const FEEDBACK_REASONS = [
  "Hard to setup",
  "Something's not working",
  "Missing features",
  "Not compatible with my setup",
  "Other",
];
```

- A Polaris `Modal` titled `Share your feedback`, with a responsive `Grid` for the checkboxes and a Polaris `TextField multiline={4}` for the optional input.
- `handleFeedbackSend` that builds the exact three-line-maximum message defined in the approved spec, calls `triggerMessage(REVIEW_QUERY, user, message)`, then resets and closes the modal. Do not include the star rating.
- `handleFeedbackClose(closeMethod)` that resets and closes without calling Crisp.

Keep all inputs optional. Keep the card mounted after every lower-rating outcome.

**Step 6: Run the lower-rating tests to verify they pass**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx
```

Expected: PASS for modal opening, Crisp formatting, Send, Cancel, and close-button cases.

**Step 7: Commit the lower-rating feedback flow**

```powershell
git add components/home/feedbackCard.jsx tests/home/FeedbackCard.test.jsx
```

### Task 3: Specify the failing five-star Shopify review tests

**Files:**
- Modify: `tests/home/FeedbackCard.test.jsx`
- Modify: `components/home/feedbackCard.jsx`
- Reference: `pages/settings/index.jsx:114`
- Reference: `pages/_document.js:8-9`
- Reference: `docs/superpowers/specs/2026-09-23-dashboard-review-feedback-design.md:95-124`

**Step 1: Write the successful review-request test**

Set `mocks.reviewRequest.mockResolvedValue({ success: true, code: "success", message: "Review modal shown successfully" })`. Click `Rate BUCKS 5 out of 5 stars` and assert:

```js
expect(mocks.reviewRequest).toHaveBeenCalledTimes(1);
expect(window.open).not.toHaveBeenCalled();
expect(screen.getByText("How is your experience with BUCKS?")).toBeInTheDocument();
```

**Step 2: Write declined-response fallback tests**

Use `it.each` for every documented decline code:

```js
[
  "already-open",
  "already-reviewed",
  "annual-limit-reached",
  "cancelled",
  "cooldown-period",
  "merchant-ineligible",
  "mobile-app",
  "open-in-progress",
  "recently-installed",
]
```

For each code, resolve the request with `{ success: false, code, message: "Unavailable" }`, click five stars, then assert the App Store URL is opened in a new tab:

```js
expect(window.open).toHaveBeenCalledWith(
  "https://apps.shopify.com/bucks-currency-converter?#modal-show=WriteReviewModal",
  "_blank"
);
```

Also verify the card remains rendered.

**Step 3: Write a request-error fallback and duplicate-request test**

- Mock `reviewRequest` to reject, click five stars, and assert the fallback opens and the card remains visible.
- Return a pending promise, click five stars twice, and assert `reviewRequest` runs once. Resolve it after the assertion to avoid an open test handle.

**Step 4: Run the five-star tests to verify failure**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx
```

Expected: FAIL because `FeedbackCard` has not yet requested Shopify Reviews or handled fallback responses.

**Step 5: Implement the five-star review flow**

In `components/home/feedbackCard.jsx`:

- Import `useAppBridge` from `@shopify/app-bridge-react` and call it inside `FeedbackCard`, matching existing usage in `pages/settings/index.jsx`.
- Add `isReviewRequestInProgress` state and return early from the five-star handler while it is true.
- Call `await shopify.reviews.request()` only for rating five.
- If `result.success` is false, call the fallback opener for every response code, without a code allowlist.
- If the request throws, call the fallback opener with `request-error` analytics context.
- Use `window.open(APP_STORE_REVIEW_URL, "_blank")`; do not navigate the embedded app's current tab.
- Clear the in-progress state in `finally`.

Keep one-to-four-star handling separate and do not open the custom feedback modal for rating five.

**Step 6: Run the five-star tests to verify they pass**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx
```

Expected: PASS for successful requests, every declined code, request errors, duplicate prevention, and card visibility.

**Step 7: Commit the Shopify review flow**

```powershell
git add components/home/feedbackCard.jsx tests/home/FeedbackCard.test.jsx
```

### Task 4: Add PostHog event coverage and implementation

**Files:**
- Modify: `tests/home/FeedbackCard.test.jsx`
- Modify: `components/home/feedbackCard.jsx`
- Reference: `components/common/AnalyticsBanner/index.jsx:25-47`
- Reference: `components/common/ThemeSyncCard.jsx:70`
- Reference: `docs/superpowers/specs/2026-09-23-dashboard-review-feedback-design.md:153-174`

**Step 1: Write failing analytics tests**

Set `window.posthog = { capture }` and verify these events and properties:

- A three-star click captures `Feedback Card - Star Selected` with `{ myshopify_domain: user.myshopify_domain, rating: 3 }`.
- Opening lower-rating feedback captures `Feedback Card - Feedback Modal Opened` with the same rating.
- Send captures `Feedback Card - Feedback Submitted` with `selected_reasons` and Boolean `has_written_feedback`; assert no free-text value appears in any capture call.
- Cancel and close-icon tests capture `Feedback Card - Feedback Cancelled` with the appropriate `close_method`.
- Five-star successful request captures `Feedback Card - Shopify Review Requested` and `Feedback Card - Shopify Review Result` with `success`, `code`, and `message`.
- A declined request captures `Feedback Card - App Store Fallback Opened` with its decline `reason`.
- A thrown request captures fallback with `reason: "request-error"`.
- The mocked `ClosePopover` dismiss control remains a separate concern; do not claim `Feedback Card - Dismissed` is implemented until `ClosePopover` offers a success callback or the card wires one without changing current persistence semantics.

Add one test where `window.posthog` is absent and verify rating, modal, and Shopify actions do not throw.

**Step 2: Run analytics tests to verify failure**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx
```

Expected: FAIL because feedback-card analytics have not been implemented.

**Step 3: Implement a local safe tracking helper**

Add a small `trackEvent(eventName, properties = {})` function in `components/home/feedbackCard.jsx` that:

```js
const { posthog } = window || {};
posthog?.capture?.(eventName, {
  myshopify_domain: user?.myshopify_domain,
  ...properties,
});
```

Guard `window` for SSR. Call this helper exactly at the tested behavior boundaries. Do not send modal free text to PostHog.

**Step 4: Add successful-dismiss analytics without weakening persistence**

Modify `components/common/closePopover.jsx` to accept an optional `onDismissed` callback. Invoke it only after `dismiss()` returns `true`; do not invoke it when persistence fails. Keep existing consumers unchanged by making the prop optional.

Pass `onDismissed={() => trackEvent("Feedback Card - Dismissed")}` from `FeedbackCard` to `ClosePopover`. Add/extend a test in `tests/common/closePopover.test.jsx` showing that the callback runs after a successful persistence response and not after a failed response.

**Step 5: Run component tests to verify analytics pass**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx tests/common/closePopover.test.jsx
```

Expected: PASS for all review-card and close-popover analytics assertions.

**Step 6: Commit analytics support**

```powershell
git add components/home/feedbackCard.jsx components/common/closePopover.jsx tests/home/FeedbackCard.test.jsx tests/common/closePopover.test.jsx
```

### Task 5: Perform integration verification and visual comparison

**Files:**
- Verify: `components/home/feedbackCard.jsx`
- Verify: `components/common/closePopover.jsx`
- Verify: `pages/index.jsx:515-516`
- Verify: `docs/superpowers/specs/2026-09-23-dashboard-review-feedback-design.md`
- Reference: [Dashboard card Figma](https://www.figma.com/design/zWmT6pWcnApOQrtvHv3KC5/UFE-Widget---UFE-3.0--ALPHA-2.0?node-id=15426-40884&t=9oNOFO8taGndXOgN-4)
- Reference: [Feedback modal Figma](https://www.figma.com/design/zWmT6pWcnApOQrtvHv3KC5/UFE-Widget---UFE-3.0--ALPHA-2.0?node-id=15426-42089&t=9oNOFO8taGndXOgN-4)

**Step 1: Run targeted tests**

Run:

```powershell
yarn vitest run tests/home/FeedbackCard.test.jsx tests/common/closePopover.test.jsx tests/api/dashboardSection.test.js
```

Expected: PASS.

**Step 2: Run the full test suite**

Run:

```powershell
yarn test
```

Expected: PASS. Investigate and fix all failures caused by this feature before proceeding.

**Step 3: Run production build verification**

Run:

```powershell
yarn build
```

Expected: Next.js production build completes successfully. Resolve import, SSR `window`, Polaris, or App Bridge errors before merge.

**Step 4: Manually verify in embedded Shopify Admin**

Start the app through the repository's normal Shopify development flow, open the Bucks dashboard, and compare it to both Figma references. Verify:

- card title, helper copy, five-star layout, and overflow menu match the card design;
- modal title, checkbox choices/order, textarea label/placeholder, and Cancel/Send footer match the modal design;
- narrow viewport stacks checkbox columns with no horizontal overflow;
- one-to-four-star path opens the custom modal, sends silently to Crisp, and retains the card;
- five-star path requests Shopify's native review modal;
- force or observe a declined/error Reviews API result and confirm the App Store URL opens in a new tab;
- only explicit overflow-menu Dismiss hides the card and it remains hidden after dashboard refresh.

**Step 5: Inspect final changes before commit**

Run:

```powershell
git status --short
```

Expected: no whitespace errors, no unrelated files, no free-text feedback in PostHog calls, and no changes under `widgets/`.

**Step 6: Commit final verification fixes if needed**

```powershell
git add components/home/feedbackCard.jsx components/common/closePopover.jsx tests/home/FeedbackCard.test.jsx tests/common/closePopover.test.jsx
```
