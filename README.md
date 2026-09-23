# Dashboard Review Feedback Design Spec

**Date:** 2026-09-23
**Status:** Approved design; awaiting written-spec review
**Surface:** Bucks merchant admin dashboard
**Audience:** Bucks frontend engineers who know React, Next.js, Polaris, and Shopify embedded apps but are new to this dashboard feedback flow.

## Problem

The dashboard's current feedback card asks merchants to choose Good or Bad. A positive response opens the Shopify App Store review page directly, while a negative response opens Crisp with a generic message. It does not provide Shopify's native review experience, structured feedback, or event-level product analytics.

## Goals

- Replace the current Good/Bad dashboard card with the approved star-rating card.
- Request Shopify's native review modal when a merchant selects five stars.
- Open the Shopify App Store review link in a new tab when Shopify cannot display its review modal.
- Collect structured feedback for one-to-four-star selections and send it silently to Crisp.
- Keep the card visible unless the merchant explicitly dismisses it from its overflow menu.
- Track the review and feedback flow in PostHog.
- Match the supplied Figma designs with existing Polaris components and no new UI library.

## Non-Goals

- Do not change storefront widget code under `widgets/`.
- Do not store feedback in a new database model or add an API endpoint for it.
- Do not add a rating field to the custom feedback modal or Crisp payload.
- Do not automatically dismiss the dashboard card after a rating selection, feedback action, Shopify review response, or fallback link.
- Do not replace the current persisted `showFeedbackSection` dismissal mechanism.

## Design References

Use these Figma nodes as the implementation reference:

- Dashboard review card: [Figma node 15426:40884](https://www.figma.com/design/zWmT6pWcnApOQrtvHv3KC5/UFE-Widget---UFE-3.0--ALPHA-2.0?node-id=15426-40884&t=9oNOFO8taGndXOgN-4)
- Custom feedback modal: [Figma node 15426:42089](https://www.figma.com/design/zWmT6pWcnApOQrtvHv3KC5/UFE-Widget---UFE-3.0--ALPHA-2.0?node-id=15426-42089&t=9oNOFO8taGndXOgN-4)

The selected section identifies the approved card as a full-width 112px dashboard card with an overflow menu, title, helper text, and five 20px outlined stars. The feedback modal reference is a 657px desktop dialog with two checkbox columns, a textarea, and Cancel/Send footer actions. Use Polaris responsive behavior for narrow screens rather than fixed pixel widths.

## Existing Integration Points

- The dashboard renders the card at `pages/index.jsx` through `FeedbackCard`.
- The current card is `components/home/feedbackCard.jsx`.
- `components/common/closePopover.jsx` persists an explicit card dismissal through `POST /api/v1/user/dashboardSection` with `section: "showFeedbackSection"` and `value: false`.
- The endpoint allows `showFeedbackSection`, and `pages/index.jsx` hides the card only when `user.showFeedbackSection === false`.
- Crisp is initialized in `pages/index.jsx` and supports a silent `REVIEW_QUERY` trigger through `utils/extras/crispChat.js`.
- The dashboard uses client-side `window.posthog.capture` when PostHog is available.

## Components

### FeedbackCard

Keep `components/home/feedbackCard.jsx` as the feature owner. It renders:

- A full-width `Grid.Cell` with a Polaris `Card`.
- Title: `How is your experience with BUCKS?`
- Helper text: `Rate us by clicking on the stars.`
- Five keyboard-accessible star buttons using Polaris star iconography. The visual state is outlined by default and fills through the selected star on hover/focus/selection.
- The existing `ClosePopover` overflow-menu control configured with `showFeedbackSection`.
- The custom feedback `Modal` only while the merchant has selected one to four stars.

The component owns transient rating-request and feedback-modal state. It must not add new user/settings persistence for a selected rating, feedback draft, or Shopify API response.

### Custom Feedback Modal

Use a Polaris `Modal` matching the Figma structure:

- Title: `Share your feedback`
- Close icon and Cancel action: close the modal without sending feedback.
- Five independent, optional checkboxes arranged in two columns on desktop and one column on narrow layouts:
  - `Hard to setup`
  - `Something's not working`
  - `Missing features`
  - `Not compatible with my setup`
  - `Other`
- Optional text area label: `What can we improve?`
- Placeholder: `We read every piece of feedback`
- Secondary Cancel action and primary Send action.

The modal has no star display and does not collect or send the selected star value.

## Behavior

### One to Four Stars

1. Merchant selects any star from one through four.
2. Capture the star-selection PostHog event.
3. Open the custom feedback modal.
4. Merchant may choose zero or more reasons and optionally enter free text.
5. Send creates a formatted Crisp message containing selected reasons and optional text only. It uses `triggerMessage(REVIEW_QUERY, user, message)`, which sends the message without opening Crisp chat.
6. After the send call, close and reset the modal form. Leave the dashboard card visible.
7. Cancel or the modal close icon closes and resets the modal without sending. Leave the dashboard card visible.

The Send button remains enabled when both the reason selection and free-text field are empty because all feedback inputs are optional.

### Five Stars

1. Merchant selects the fifth star.
2. Capture star selection and Shopify review request events.
3. Call `await shopify.reviews.request()` in the embedded-app client.
4. If the response is successful, record the successful result. Shopify owns the review modal from this point.
5. If the response is unsuccessful, record its `code` and `message`, then open this URL in a new browser tab:

```txt
https://apps.shopify.com/bucks-currency-converter?#modal-show=WriteReviewModal
```

6. If the request throws, record an error outcome and open the same URL in a new browser tab.
7. Leave the dashboard card visible for every outcome.

Supported declined response codes are:

```txt
already-open
already-reviewed
annual-limit-reached
cancelled
cooldown-period
merchant-ineligible
mobile-app
open-in-progress
recently-installed
```

Fallback applies to every declined code and any thrown request error. It must not be restricted to a hand-maintained subset of codes, so newly introduced declined codes continue to receive the fallback.

### Explicit Card Dismissal

Only the existing overflow menu's `Dismiss` action hides the card. It persists `showFeedbackSection: false` through the existing dashboard-section endpoint. On a future dashboard render, `pages/index.jsx` continues to suppress the card from `user.showFeedbackSection`.

Do not add dismissal behavior to:

- Star buttons
- Feedback-modal Send
- Feedback-modal Cancel or close icon
- Shopify review success
- Shopify review decline
- App Store fallback-link opening

## Crisp Payload

The message must make feedback readable in Crisp without exposing a rating. Use a stable text format:

```txt
Dashboard feedback
Reasons: Hard to setup, Missing features
What can we improve: The initial configuration is unclear.
```

Omit the `Reasons:` line when no reason is selected. Omit the `What can we improve:` line when no text is entered. If both are empty, send `Dashboard feedback` alone.

The message uses `REVIEW_QUERY`, not `SENT_QUERY`, so the merchant stays on the dashboard and the Crisp messenger does not open automatically.

## PostHog Analytics

Capture client-side events only when `window.posthog.capture` is available. Every event includes:

```js
{ myshopify_domain: user?.myshopify_domain }
```

Required events and properties:

| Event | Additional properties | Trigger |
| --- | --- | --- |
| `Feedback Card - Star Selected` | `rating` | Any star click |
| `Feedback Card - Feedback Modal Opened` | `rating` | One-to-four-star path opens the custom modal |
| `Feedback Card - Feedback Submitted` | `selected_reasons`, `has_written_feedback` | Send feedback to Crisp |
| `Feedback Card - Feedback Cancelled` | `close_method` (`cancel` or `close_icon`) | Cancel or modal X |
| `Feedback Card - Shopify Review Requested` | none | Before calling Shopify Reviews API |
| `Feedback Card - Shopify Review Result` | `success`, `code`, `message` | Shopify API resolves |
| `Feedback Card - App Store Fallback Opened` | `reason` | Declined result or thrown request error |
| `Feedback Card - Dismissed` | none | Persisted card dismissal succeeds |

Use `request-error` for the fallback `reason` when the Reviews API rejects or throws before returning a structured result. PostHog payloads must not include the merchant's free-text feedback.

## Error Handling and Accessibility

- Prevent duplicate five-star review requests while a request is in progress.
- If opening the fallback tab is blocked by the browser, do not dismiss the card; log the fallback event before attempting to open it.
- Keep the feedback modal usable by keyboard: focus enters the dialog, checkbox and textarea controls have labels, Escape/close behavior follows Polaris modal defaults, and focus returns to the initiating star after close.
- Give every star button an accessible label such as `Rate BUCKS 3 out of 5 stars`.
- Use Polaris layout components for responsive checkbox columns. Avoid horizontal overflow on mobile.
- Keep analytics and Crisp failures non-blocking: the UI must remain usable and the card must remain visible if either integration fails.

## Acceptance Criteria

- `FeedbackCard` replaces Good/Bad controls with the Figma-based five-star review card.
- The card remains at the existing dashboard placement in `pages/index.jsx`.
- Selecting stars one through four opens a custom modal with only the five specified checkboxes and optional text area; it contains no rating UI.
- Sending feedback calls the silent Crisp `REVIEW_QUERY` path with no rating in the payload and closes the modal.
- Cancelling or closing the modal sends nothing and leaves the card visible.
- Selecting five stars calls Shopify's `shopify.reviews.request()` API.
- Any Shopify declined result or thrown request error opens the approved App Store review URL in a new tab.
- The card hides only after its explicit overflow-menu Dismiss action persists successfully.
- All required PostHog events use the specified names and do not include free-text feedback.
- No new dependency, API route, database field, or widget change is introduced.

## Verification

- Unit-test star routing: ratings one through four open feedback; five requests Shopify review.
- Unit-test Crisp message formatting for reasons-only, text-only, both, and empty feedback.
- Mock Reviews API success, each declined response code, and rejection; verify fallback behavior only for declines/errors.
- Test close, Cancel, Send, successful review request, declined review request, and fallback opening all leave the card visible.
- Test persisted overflow-menu dismissal and dashboard re-render suppression.
- Run the relevant lint and test commands defined in `package.json` before merging.
