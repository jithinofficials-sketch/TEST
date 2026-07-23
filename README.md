# Onboarding Completion Modal Design Spec

**Date:** 2026-07-24
**Status:** Draft for review
**Scope:** Fix when the onboarding completion modal appears, persists through refresh, and stops after dismissal.

## Problem

The onboarding completion modal can show at the wrong time and can show again after it was already shown once.

There are two root causes:

1. `banners.onboardingBanner` has unclear meaning. The app currently treats `true` as "show the modal on page load", but closing the modal writes `false`, which makes an already-dismissed modal look the same as a modal that was never shown.
2. Onboarding step completion can be calculated from stale persisted data instead of the current live status. For example, `moneyFormat` can be disabled in Shopify, but old DB/global state can still contain `moneyFormat: true`. Completing another step can then make the app think all three steps are complete.

## Goals

- Show the completion modal only when onboarding changes from incomplete to complete.
- Treat onboarding as complete only when all three named steps are currently true:
  - `moneyFormat === true`
  - `themeExtension === true`
  - `appEnabled === true`
- Do not show the modal to existing users who already have `onboardingCompleted: true`.
- If the modal is open and the user refreshes the page, show it again until the user dismisses it.
- Once the modal is dismissed, do not show it again for that install.
- After uninstall/reinstall, allow the modal to show again after onboarding is completed.
- Keep the fix minimal and avoid a Prisma schema change.

## Non-Goals

- Do not add a new database field unless the current flags cannot satisfy the behavior.
- Do not migrate historical banner values.
- Do not redesign the modal UI.
- Do not change onboarding step labels, copy, or layout.

## Decisions

### Existing Completed Users

Users who already have `onboardingCompleted: true` must not see the completion modal, even if `banners.onboardingBanner` is `false`.

Reason: historical data cannot reliably distinguish "modal never shown" from "modal already dismissed", because both can be represented by `false` today.

### Modal Pending State

Use `banners.onboardingBanner` as a pending modal flag:

- `true`: completion modal is pending and should open on page load.
- `false` or missing: completion modal is not pending.

This works because `onboardingCompleted` becomes the one-time guard. A modal can only be scheduled while the user was previously incomplete.

### Dismissal

Closing the modal must set `banners.onboardingBanner` to `false`.

This includes both modal exits:

- normal close
- "No, I need help"

After either action, the modal must not show again until reinstall.

### Reinstall

The uninstall webhook must reset:

- `onboardingCompleted: false`
- `onboardingProgress.step.appEnabled: false`
- `banners.onboardingBanner: false`

This allows a reinstalled user to complete onboarding again and see the modal again.

## Current Files Involved

- `pages/index.jsx`
  - Owns dashboard page state.
  - Opens the completion modal on page load when the modal is pending.
  - Passes `setShowCompletionPopup` into onboarding components.
- `components/home/OnboardingCompletionPopup.jsx`
  - Renders the modal.
  - Handles close and "No, I need help" actions.
  - Must persist dismissal by setting `banners.onboardingBanner` to `false`.
- `components/home/onBoardingBanner.jsx`
  - Runs live checks for money format, theme extension, and app status.
  - Maintains `onBoardingObj`, the current frontend view of the three step states.
- `components/common/onBoardingCard.jsx`
  - Saves onboarding progress after user actions.
  - Currently uses stale global/DB state as the save base.
  - Must use current live step state as the save base.
- `pages/api/v1/user/onboarding.js`
  - Merges incoming onboarding step data into DB.
  - Sets `onboardingCompleted: true` when all three named steps are true.
- `pages/api/v1/banner/dismiss.js`
  - Updates banner flags.
  - Existing behavior is sufficient.
- `utils/webhooks/app_uninstalled.js`
  - Resets install-specific onboarding state on uninstall.

## Proposed Behavior

### First-Time Completion

1. User starts with `onboardingCompleted: false`.
2. User completes onboarding steps in any order.
3. After each confirmed action, the app saves all three current step statuses to DB.
4. When the action causes all three statuses to become true, the app:
   - sets `onboardingCompleted: true` through `/api/v1/user/onboarding`
   - sets `banners.onboardingBanner: true` through `/api/v1/banner/dismiss`
   - opens the completion modal
5. If the user refreshes before closing, the dashboard sees `onboardingBanner: true` and opens the modal again.
6. When the user closes the modal or clicks "No, I need help", the app sets `onboardingBanner: false`.
7. Future refreshes do not open the modal.

### Existing Completed User

1. User already has `onboardingCompleted: true`.
2. Dashboard must not schedule a new modal for that user.
3. If this user later toggles onboarding-related settings, the modal still must not show because the user was already completed before this fix.

### Reinstalled User

1. User uninstalls the app.
2. Uninstall webhook resets `onboardingCompleted: false`, app step state, and `onboardingBanner: false`.
3. User reinstalls the app.
4. Once all three live statuses become true again, the modal can show once.

## Completion Calculation

All completion checks must use named keys:

```js
const isOnboardingComplete = (step = {}) =>
  step.moneyFormat === true &&
  step.themeExtension === true &&
  step.appEnabled === true;
```

Avoid count-based checks:

```js
Object.values(step).filter(value => value === true).length === 3
```

Count-based checks are unsafe because they can pass with stale or unexpected keys.

## Save Payload Rule

When saving onboarding progress, use the current live state as the source of truth, not stale DB/global state.

The payload should start from `onBoardingObj`:

```js
const payload = {
  moneyFormat: onBoardingObj?.moneyFormat === true,
  themeExtension: onBoardingObj?.themeExtension === true,
  appEnabled: onBoardingObj?.appEnabled === true,
};
```

Then apply the confirmed action:

```js
if (value === 1) {
  payload.moneyFormat = true;
} else if (value === 2) {
  payload.themeExtension = true;
} else if (value === 3) {
  payload.appEnabled = true;
}
```

This ensures false statuses are saved too. For example, if money format is currently disabled and app status is completed, the saved payload remains:

```js
{
  moneyFormat: false,
  themeExtension: true,
  appEnabled: true
}
```

The modal will not show in that state.

## Modal Scheduling Rule

The completion modal can be scheduled only when all conditions are true:

```js
completed === true &&
wasAlreadyCompleted !== true &&
popupAlreadyPending !== true
```

Where:

- `completed` means all three named onboarding steps are true after the current save.
- `wasAlreadyCompleted` is read before saving the current action from `userDataGlobal?.onboardingCompleted ?? userData?.onboardingCompleted`.
- `popupAlreadyPending` is read from `userDataGlobal?.banners?.onboardingBanner ?? userData?.banners?.onboardingBanner`.

If the conditions pass:

1. Call `/api/v1/banner/dismiss` with `{ bannerType: "onboardingBanner", value: true }`.
2. Update global user state so `banners.onboardingBanner` is true.
3. Open the modal.
4. Fire confetti.

## Page-Load Rule

`pages/index.jsx` should open the modal on page load only when:

```js
user?.onboardingCompleted === true &&
user?.banners?.onboardingBanner === true
```

This supports refresh while the modal is pending. It does not create a new modal for existing completed users because only the completion transition should set the pending flag.

## Dismiss Rule

`OnboardingCompletionPopup` should dismiss by writing:

```js
{ bannerType: "onboardingBanner", value: false }
```

This applies to both:

- close button / modal close
- "No, I need help"

After the API call, close the modal locally.

If the dismissal API fails, still close the modal locally but log the error. A failed API call may cause the modal to reappear on refresh, which is acceptable because the DB did not confirm dismissal.

## Error Handling

- If `/api/v1/user/onboarding` fails, do not show the modal.
- If setting `onboardingBanner: true` fails, do not show the modal because refresh persistence would not be guaranteed.
- If dismissal fails, close locally and log the error.
- Existing Sentry/PostHog tracking can remain unchanged.

## Testing Plan

Manual checks are required because these flows depend on Shopify live state and DB state:

1. Fresh user completes steps in order: money format, theme extension, app enabled. Modal shows once.
2. Fresh user completes steps out of order. Modal shows when the final missing step becomes true.
3. Money format is disabled while old DB state has `moneyFormat: true`; user completes app status. Modal does not show.
4. Modal is open; user refreshes. Modal opens again.
5. User closes modal. Refresh does not open it again.
6. User clicks "No, I need help". Refresh does not open it again.
7. Existing user with `onboardingCompleted: true` and `onboardingBanner: false` does not see modal.
8. Existing user with `onboardingCompleted: true` and historical `onboardingBanner: true` opens pending modal on refresh, then dismissal clears it.
9. User uninstalls and reinstalls; after all three steps are complete again, modal can show once.

Automated tests should cover pure logic where practical:

- named-key completion helper returns true only when all three expected keys are true.
- modal scheduling blocks users who were already completed.
- modal scheduling allows a previously incomplete user who just completed all three steps.

## Risks

- Historical users with `onboardingCompleted: true` and `onboardingBanner: true` may see the modal once on page load. This is consistent with the pending-modal interpretation, but if this historical state is common and unwanted, the page-load rule can also require a new transition marker. That would require a new DB field or migration.
- If live checks fail due to Shopify API errors, onboarding should stay incomplete rather than showing the modal.
- Some existing code mutates `userDataGlobal` directly in settings-related pages. This spec avoids changing those broader patterns to keep the fix minimal.

## Acceptance Criteria

- The completion modal appears exactly once per install after the user completes all three live onboarding steps.
- The modal does not appear when any of the three live steps is false.
- Existing users with `onboardingCompleted: true` and dismissed modal state do not see the modal again.
- Refresh keeps the modal open while `onboardingBanner: true`.
- Closing the modal or clicking "No, I need help" sets `onboardingBanner: false` and prevents future display.
- Reinstall resets enough state for the modal to show again after onboarding is completed.
