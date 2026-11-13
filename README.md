  # RFC: Session-Based Support Chat Prompts

  - Status: Draft - awaiting confirmation
  - Author: Engineering draft
  - Reviewers: BUCKS product and support owners
  - Type: #feature-rfc
  - Audience: BUCKS merchants in admin; support agents receiving context

  ## 1. Problem

  Position edits, enabling Cart notification, and repeatedly rechecking an inactive app embed can indicate a need for setup help. Opening chat for every trigger interrupts the merchant. The current once-per-store flags also prevent assistance on a later visit and do not capture subsequent setup actions for support.

  ## 2. Why This Matters

  Offer help after a short pause, limit interruptions to one automatic opening per visit, and give support context about subsequent actions. Success means one automatic opening at most per session, no opening before three seconds, and later events reaching support without reopening chat.

  ## 3. Proposed Solution

  | Action | Proposed behavior |
  | --- | --- |
  | Change any Desktop or Mobile position option | Wait three seconds after the latest change, then process the placement trigger. |
  | Successfully save Cart notification from off to on | Wait three seconds after the successful save, then process the notification trigger. |
  | Click the app embed banner's Recheck status button three times within a rolling 60-second window while its status is disabled | Wait for the third check to complete. Only if it confirms the embed is still disabled, wait three seconds and process the app-embed trigger. |
  | First eligible trigger in the session | Show the agreed support greeting and open Crisp. |
  | Further triggers in any of the three scenarios | Record context for support without showing another greeting or opening chat. |

  Keep the existing greetings. Add no merchant-facing settings. Typing or repeated option changes restart the placement timer, so one burst produces one event. Cancel pending triggers when leaving their section, unmounting, or hiding the app. Do not deliver stale prompts on return. Staff impersonation remains excluded.

  ### App embed recheck scenario

  The merchant confirmed that the trigger must wait for the check result; a disabled status displayed before the check is not enough.

  - Count only manual Recheck status actions accepted while the banner shows disabled. Automatic mount checks and clicks ignored while a check is pending do not count.
  - Use click timestamps: three accepted clicks must fall within 60 seconds of one another. The third response may arrive after that window; the click timestamps determine eligibility.
  - If that check returns disabled, schedule the prompt three seconds after the response. If the embed becomes enabled before delivery, cancel it and clear the click history.
  - Failed, timed-out, or unknown checks cannot qualify the trigger. Proposed conservative handling: clear the click history on a failed check, so stale disabled status cannot accumulate toward a prompt.
  - After processing the threshold, clear the click history. A fourth click alone must not create another event; a new group of three is required. Clear history when leaving/unmounting the banner or hiding the app.
  - This scenario shares the same one-opening-per-session allowance as placement and cart notification. If another scenario already opened chat, send app-embed context only.

  Suggested greeting (pending copy approval):

  > Hi! Is your app embed still showing as inactive after rechecking? I'm happy to help you activate it in your theme and get your currency converter working!

  ### What counts as a session? (Proposed, not approved)

  Three options for review; choose one before implementation:

  | Option | How we detect a new session | Return/refresh behavior | Tradeoff |
  | --- | --- | --- | --- |
  | **1. 30-minute inactivity (recommended)** | Store a shop-scoped session ID and last meaningful activity timestamp; expire after 30 minutes. | Refresh and brief returns keep the session. Returning after expiry starts a new one. Coordinate across tabs in the same browser. | Best protection against repeated interruptions; requires activity tracking. |
  | **2. New browser-tab visit** | Keep a shop-scoped app-session ID in sessionStorage. | Navigation and refresh normally retain it; a fresh independent tab normally starts a new session. Restored/duplicated tabs can retain or copy sessionStorage. | Simpler, but separate tabs can each open chat; browser close is not a reliable reset signal. |
  | **3. Every return to the app** | Treat a hidden-to-visible transition as a new visit, and a fresh document load as a new visit. | A brief tab switch or refresh can restore eligibility. Switching sections inside the visible app does not. | Most aggressive; returning from the theme editor can open help again. Visibility detects a return, not the merchant's intent. |

  The remaining session design below assumes option 1; options 2 or 3 would change session storage, cross-tab scope and reset tests. No option creates a new Crisp conversation automatically: app sessions and Crisp conversation sessions are different.

  Use a 30-minute inactivity window, scoped to the store and browser. Activity means meaningful interaction with the visible BUCKS app, such as a pointer action, keypress, or navigation; background polling does not count.

  - Refreshing, changing sections, closing Crisp, or briefly switching tabs does not reset the session.
  - Returning before 30 minutes continues the session; returning after 30 minutes starts a new one.
  - Keep the same session across BUCKS tabs in the same browser. Different browsers/devices are separate sessions; this is not a Shopify authentication session.
  - Persist session ID and last activity locally, with throttled activity writes. Check expiry before refreshing the activity timestamp on return.

  This is preferable to detecting a browser close, which is unreliable, or treating every focus event as a new visit, which causes repeated openings.

  ### Support context: simplest option versus internal notes

  **Minimal alternative:** use Crisp session events for subsequent actions, plus session data for the latest trigger context. The Web SDK documents `Crisp.session.pushEvent()` and `Crisp.session.setData()`. Before release, confirm where these appear for agents in our Crisp workspace and whether they cause any existing automation to open chat. These are context records, not private conversation notes; use this alternative only with product/support approval.

  **Recommended proposal for the requested notes:** use a backend integration to send private conversation notes. Crisp's REST API documents sending a message with type `note` through `POST /v1/website/{website_id}/conversation/{session_id}/message`. The documented fields support `from: operator`, `origin: chat`, and string content. This makes a server-side note integration possible; account permissions and agent/merchant visibility must still be verified in our workspace before rollout. This is a recommendation pending approval, not a verified production integration.

  #### How subsequent triggers reach the agent

  1. After the three-second delay, the shared coordinator checks the app session's opening allowance. If already consumed, it selects context-only delivery, including for a repeat in the same section.
  2. For the minimal event option, send a named Crisp session event and update latest-context session data. Never call chat open or show another local greeting on this path. Check existing Crisp automations so an event cannot indirectly reopen chat.
  3. For the note option, the browser submits a bounded trigger type, section, app-session ID and unique event ID to an authenticated BUCKS endpoint. The backend builds the note from allowed fields; it does not accept arbitrary note text or trust a browser-supplied conversation ID.
  4. Resolve the authenticated store's existing Crisp conversation using its persisted continuity token and Crisp's documented Get Session Identifier From Token endpoint, or an independently verified server-side mapping. The local app-session ID is not a Crisp conversation ID. Wait for token persistence when initialization just created it; missing mappings must not create unrelated conversations.
  5. Using server-only Crisp API credentials with the required access, post the note to that conversation. Deduplicate event IDs and coalesce rapid identical actions. Do not change the visitor chat-open state or send a visitor/public text message.
  6. If delivery fails, keep the merchant's settings flow working and log the failure. Retry known transient failures with bounded backoff; ambiguous delivery timeouts need duplicate handling, not blind retries. Failure never restores permission to open chat again.

  Example internal note:

  > BUCKS setup activity: Cart notification was successfully enabled. Placement help already opened during this app session, so no additional popup was shown. Section: Advanced. Time: [UTC timestamp].

  **Important limitation:** showing a local greeting and opening Crisp does not prove that a merchant has started an inbox-visible conversation. Crisp documents that newly created conversations are not inbox-visible until a visitor message is sent. Verify note visibility for an existing conversation with no visitor reply; do not fabricate a visitor message to force inbox visibility. A note also does not guarantee an agent notification. These are explicit acceptance checks for the note option.

  Suggested context: trigger type, section, changed field names, timestamp and app-session ID. For app-embed help, include `app_embed_recheck`, manual recheck count (3), window duration (60 seconds), and confirmed result (disabled). Do not transmit custom selector contents or merchant-entered notification text. Debounce rapid edits and coalesce identical events within a short interval to avoid flooding the agent.

  ## 4. End-to-End Flow

  ```text
  Merchant changes position OR successfully saves notification off -> on
    OR third manual embed recheck within 60 seconds confirms disabled
    -> wait 3 seconds (restart placement timer on another edit)
    -> check active section and current app session
    -> initialize Crisp and record trigger context
    -> session already used its automatic opening?
        yes: authenticated backend -> private Crisp note; no chat opening
        no: atomically reserve opening, show greeting, open chat
  ```

  If Crisp is already open, record context and mark the session as assisted without showing another greeting. This prevents reopening it after the merchant closes it. Concurrent triggers share one opening allowance; the other event becomes context only. No shopper-facing behavior changes.

  ## 5. Implementation Details

  - **Frontend:** reuse the existing Display edit handler, Advanced successful-save handler, app embed banner's manual recheck/result handling, and Crisp helper. Add a small shared session/timer coordinator and a short rolling click history in the banner; keep widget code isolated. Keep the support-delay timer separate from the banner's existing check-timeout timer.
  - **Note delivery:** add an authenticated backend path that validates the store, resolves its Crisp conversation, deduplicates the event and posts a private note using server-only credentials. Record delivery outcome separately from the session's opening allowance. If conversation resolution or note delivery is unavailable, report the failure internally without opening chat or silently substituting a public message. Events/session data remain an explicitly approved alternative, not an automatic fallback.
  - **Persistence:** replace permanent per-trigger suppression with a session-scoped atomic opening claim. Scope claims to the authenticated store plus an opaque app-session ID, and expire old claims. Browser-local state alone is insufficient for simultaneous-tab claims. Final storage design is subject to session-definition approval.
  - **Migration:** stop consulting old `supportChatTriggers` booleans for session eligibility. Leave old data untouched initially. Removing obsolete fields can be a separate cleanup.
  - **Failures:** chat or context delivery must not block saving settings. Do not mark a session assisted when initialization fails. Release known failed opening attempts; a browser crash after reservation can still consume an opening without visible delivery. Do not promise exactly-once visual delivery.
  - **Verification:** fake-timer tests for three-second delay/debounce/cancellation; off-to-on successful saves only; three manual embed rechecks within 60 seconds versus outside that window; automatic checks excluded; enabled/failed/unknown results do not trigger; fourth click does not repeat the event; session continuation/expiry; refresh and concurrent tabs; context-only subsequent triggers across all three scenarios; impersonation; failed initialization. Verify agent-visible context and no automatic reopening in the actual Crisp workspace.
  - **Delivery:** first confirm this RFC and the agent-context mechanism, then implement and test the session behavior, then verify with a merchant login and support-agent account before rollout.

  ## 6. Decisions Needed

  1. Choose session option **1 (30-minute inactivity)**, **2 (new tab visit)**, or **3 (every return)**. Option 1 is recommended; none is approved yet.
  2. Confirm Cart notification still triggers **after a successful off-to-on save**, rather than immediately on toggle?
  3. Approve the recommended **private conversation-note integration**, subject to workspace verification, or choose the smaller **events/session data** alternative?
  4. Approve replacing **once per store** with **once per session**, so returning merchants can receive help again?
  5. Approve the suggested app-embed greeting and conservative reset of recheck history after failures? Waiting for a confirmed disabled result is already approved.

  ## References

  - Crisp Web SDK events, session data and local operator messages: https://docs.crisp.chat/guides/chatbox-sdks/web-sdk/npm/
  - Crisp REST note message support: https://docs.crisp.chat/references/rest-api/v1/#send-a-message-in-conversation
  - Conversation resolution: https://docs.crisp.chat/references/rest-api/v1/#get-session-identifier-from-token
  - New conversation inbox visibility: https://docs.crisp.chat/references/rest-api/v1/#create-a-new-conversation
  - Current integration: `components/display-tab/index.jsx`, `pages/advanced/index.jsx`, `components/home/AppEmbedBanner.jsx`, `utils/extras/crispChat.js`, `pages/api/v1/user/supportPrompt.js`.
