# RFC: Support Chat A/B Experiment

- **Audience:** BUCKS developers, product reviewers, and support operators familiar with Shopify apps but new to this experiment.
- **Status:** Draft; implementation and rollout are not approved by this document.
- **Author:** AI-assisted draft based on the developer's requirements.
- **Reviewers:** BUCKS product owner, app engineering, and support operations.
- **Type:** #experiment-rfc
- **Surface:** Merchant-facing BUCKS admin; support operators are an additional affected population. No shopper surface.

## 1. Problem

BUCKS currently offers proactive help for app embed activation, converter position, and cart notifications. Eligible actions generate context events and, when allowed, a dedicated Crisp user event that activates an agent-style message. Event submission and trigger execution do not establish that the merchant replied.

The proposed experiment compares the current agent greeting with an app-generated visitor message. We need stable assignment and explicit outcome attribution, without treating an automated visitor message as genuine merchant engagement.

The anchor is **start meaningful support conversations**. Native Crisp Trigger Analytics measures saved-trigger executions, not meaningful conversations. The requested design uses six new milestone triggers, PostHog activation records, and weekly support review in Crisp to determine actual conversion. Silent milestone-trigger counting and visitor repeat limits must be validated before implementation.

## 2. Why This Matters

The current tracking records opportunities to offer help, not whether merchants engage. Without separating activation, automated sending, and human replies, variant B could appear successful simply because the app sends a message automatically.

This experiment should establish whether either approach produces more genuine merchant engagement per eligible shop, while measuring additional support workload. No baseline engagement rate or support-load estimate has been supplied; this RFC does not claim a proven conversion benefit.

## 3. Proposed Solution

### Confirmed requirements

- One stable variant per shop across all three features and visits.
- Approximately 50/50 deterministic assignment using a new experiment key, proposed as `support_chat_v1`; independent of onboarding.
- Variant A retains the agent-style greeting experience, but the greeting must not execute a counted saved trigger. Its milestone is the first actual merchant reply after activation in the same tab session.
- Variant B automatically sends a predefined message as the visitor and opens chat. Its automatic-send outcome is recorded separately from human replies.
- Create three separate A reply triggers and three separate B automatic-send triggers in Crisp. Track shop/variant/feature activations in PostHog; support reviews meaningful conversations in Crisp weekly, at 7-day and 14-day cohort windows.
- Keep current eligibility and suppression rules. Do not reset consumed position/cart records for the experiment.

### Alternatives considered

| Approach | Benefit | Tradeoff / decision |
| --- | --- | --- |
| Current greeting vs automatic visitor message | Tests the requested support-initiation difference | Selected for this draft; creates a visitor message without a merchant Send action |
| Current greeting vs prefilled composer | Merchant explicitly chooses to send; cleaner engagement comparison | Simpler alternative for support intent, but not the selected variant B |
| Six milestone triggers plus manual review | Separates A replies and B automatic sends by feature | Selected requirement; native counts are milestones, not conversions, and silent counting remains unverified |

Merchant burden: no settings or new screen. Variant B removes the Send action but may surprise merchants by speaking on their behalf and may create support work without a request. Product/support review must assess this tradeoff before launch; fewer clicks alone is not evidence that B is better.

### Message copy

| Feature | Variant B visitor message |
| --- | --- |
| Position | Trying to figure out how to position the currency converter on the store! |
| Cart notification | Yay! Cart Notification is now enabled! 🎉 |
| App embed | BUCKS app embed isn’t activating. Can you help? |

Position and embed wording follow the developer's supplied copy. Cart copy corrects the supplied "is are" typo and renders `:tada:` as an emoji; this correction remains a review item because the request for an RFC replaced final design approval. Preserve variant A's existing greeting copy when moving its display to the app-controlled path.

### Metrics

Assignment is not exposure. Record a separate activation when the app submits the variant's help action, labelled as an attempt rather than a confirmed impression.

| Proposed event | When recorded | Meaning |
| --- | --- | --- |
| `bucks_support_activation` | Once per eligible proactive-action attempt | Denominator and variant exposure attempt; not proof of display |
| A feature-specific reply event (table below) | First attributable actual merchant send in the tab session | Requests one A milestone trigger; not a conversion verdict |
| B feature-specific automatic-send event (table below) | B's automatic message is confirmed by a verified Crisp send boundary | Requests one B milestone trigger; not a human conversion |

Track activation in PostHog and preserve existing feature context/profile events. Send the feature-specific milestone event to Crisp for its corresponding On user event rule. No automated conversion verdict is inferred from either milestone. If the SDK only confirms submission, do not label B's milestone a confirmed send; resolve the measurement contract before rollout.

### Six new Crisp milestone triggers

| Variant / feature | Trigger identifier | On user event |
| --- | --- | --- |
| A / embed | `bucks-support-a-embed-reply` | `bucks_support_a_embed_reply` |
| A / position | `bucks-support-a-position-reply` | `bucks_support_a_position_reply` |
| A / cart | `bucks-support-a-cart-reply` | `bucks_support_a_cart_reply` |
| B / embed | `bucks-support-b-embed-sent` | `bucks_support_b_embed_sent` |
| B / position | `bucks-support-b-position-sent` | `bucks_support_b_position_sent` |
| B / cart | `bucks-support-b-cart-sent` | `bucks_support_b_cart_sent` |

A greeting alone emits no milestone. A's first qualifying same-tab reply requests one feature-specific trigger. B's confirmed automatic send requests one feature-specific trigger; later human replies do not request another B trigger. Deduplicate each milestone per activation.

These triggers must not add messages or reopen chat: delivery is owned by the app. **Feasibility gate:** verify with Crisp whether a saved trigger with no visible actions can execute and count in native analytics. Do not assume actionless triggers count, add unwanted visible actions to manufacture counts, or emit Crisp's internal `trigger:fired` manually. Visitor-level remembered execution may still suppress subsequent activations; this design cannot promise one native count for every tab's milestone.

Common properties: `experiment`, `variant`, `feature` (`embed`, `position`, `cart`), `activation_id`, `source`, and `message_origin` where applicable. Use the existing shop identity convention in PostHog; do not include message text or visitor email in experiment properties. Variant and experiment must be attached explicitly to every event, not assumed to be inherited.

**Conversion assessment:** support reviews linked Crisp conversations weekly against the shared anchor, meaningful support conversations. Proposed rubric: genuine merchant participation plus a substantive support exchange relevant to the help context; automated visitor messages and automated greetings alone do not qualify. Product/support must approve the rubric before launch.

Compare unique activated shops with a reviewer-confirmed meaningful conversation against unique activated shops per variant. Evaluate each cohort after a full 7 days and again after 14 days; the 14-day cumulative window includes the first week. Keep a review ledger with experiment, variant, shop, feature, activation ID/time, Crisp conversation reference, reviewer, window, and verdict. Review conversations even when a browser reply milestone is missing; native trigger counts are not the conversion denominator. B automatic-send counts and A reply counts must not be compared as equivalent conversions.

Guardrails: automatic conversations created, repeat messages, delivery failures, and support workload. Activation attempts remain distinct from confirmed display. Crisp suppression affects milestone counts even when app-controlled delivery succeeds.

## 4. End-to-End Flow

```text
Authenticated shop -> stable experiment assignment
  -> existing feature condition and eligibility checks
  -> existing context event
  -> proactive allowance unavailable: stop (context only)
  -> allowance available: create activation and install attribution listener
       A: app displays agent-style greeting and opens chat (no milestone)
          -> first actual same-tab merchant reply -> A feature milestone event
       B: submit automatic visitor message -> open chat
           -> confirmed automated send -> B feature milestone event
       Both: weekly Crisp review -> meaningful-conversation verdict at day 7/14
```

- **Merchant:** no experiment selector. A sees the existing help experience; B sees an app-submitted visitor message in chat. A manual support opening before activation is not attributed to the experiment.
- **Support operator:** B's visitor message must carry associated profile context showing `variant: B` and automated origin, so it is not mistaken for a deliberate support request.
- **Shopper:** no change to widget behavior, prices, or storefront configuration.
- **App:** eligibility, assignment, attribution, and delivery remain separate responsibilities. A profile-event write must never itself count as a merchant reply.

Attribution follows the first activation consuming the tab's allowance. Later context-only features do not replace it. Same-tab route navigation and reload should preserve pending attribution; closing the tab ends the proposed browser-only attribution scope. Messages sent on another device or after that scope ends are excluded. Session-storage copying into duplicated tabs is a known boundary requiring validation.

## 5. Implementation Details

### Reusing the onboarding approach

`docs/AB_TESTING_IMPLEMENTATION.md` describes server-side SHA-256 assignment using shop identity and an experiment key. Reuse that concept, not its unverified status claims:

- The referenced `utils/abTesting/assignVariant.js` and `onboarding_variant` usage were not found in the current code search.
- Hash bucketing produces approximately even groups across a population, not a guaranteed exact 50/50 sample.
- Fixed sample-size and time-window suggestions in that document are not sufficient for this experiment without baseline rates and a minimum effect size.
- Hash assignment alone does not create a PostHog feature-flag experiment or automatically annotate future events.

Propose a server-only assignment helper using a normalized authenticated shop domain and an unambiguous experiment-key separator. Compute the same result for Home, Settings, and Advanced, including direct page loads. Do not import Node crypto into browser or storefront bundles. No assignment database table is required; changing the key or bucketing algorithm creates a new experiment.

If shop identity or assignment is unavailable, retain the current A behavior and exclude that activation from experiment reporting rather than recording a fabricated assignment. No storefront dependency sharing.

### Delivery and integration boundaries

| Existing area | Proposed responsibility |
| --- | --- |
| `pages/index.jsx`, `pages/settings/index.jsx`, `pages/advanced/index.jsx` | Provide stable server-assigned experiment metadata on every entry path |
| `utils/extras/chatTriggers.js` | Select variant delivery within the existing eligibility/message gate; retain claims and serialization |
| `components/hooks/useChatTriggers.js` | Supply delivery adapters and current shop/experiment context |
| `utils/extras/crispChat.js` | Reuse readiness and identity continuity; add a dedicated automatic-send path if existing helpers do not fit |
| `utils/extras/crispProfileEvent.js` | Submit profile context and experiment events through the ready runtime |
| `utils/registerPostHogEvent.js` | Preserve non-blocking analytics delivery |
| `pages/api/v1/user/chatTriggers.js` | Retain authenticated position/cart claim and completion semantics |

A uses direct local agent-style message display and chat opening to avoid counting the initial greeting as a saved-trigger execution. Neither variant emits the existing `_message` activation events during the experiment. Disable or isolate the old greeting triggers for experiment traffic; retain unrelated/manual support behavior. B uses Crisp's documented visitor-message send operation and chat opening after readiness. The six new triggers are only for milestones, subject to the silent-counting feasibility gate. Verify actual send callbacks against the installed/live SDK before implementing milestone semantics.

No Prisma schema change is proposed for assignment or same-tab attribution. Existing installation consumption stays in the database. Preserve three failed manual dashboard embed checks in two minutes, three-second position/cart timing, one proactive attempt per tab, and impersonation exclusion. Existing consumed shops are not newly eligible for position/cart, so the experiment population is eligible shops, not every installation.

### Automatic Crisp segments and custom data

On an eligible experiment activation, after Crisp readiness, attach the assigned variant segment using `session:segments` without overwriting unrelated segments:

- A: `bucks-support-v1-a`
- B: `bucks-support-v1-b`

Segments group conversations for support review; they are neither conversions nor trigger counts. Verify available segment filters in the actual Crisp workspace before documenting a saved inbox view. Apply the segment on activation, not merely assignment, so it identifies participating conversations.

Set the following namespaced fields through `session:data`:

| Custom data key | Value |
| --- | --- |
| `bucks_support_experiment` | `support_chat_v1` |
| `bucks_support_variant` | `A` or `B` |
| `bucks_support_feature` | `embed`, `position`, or `cart` |
| `bucks_support_activation_id` | Current activation ID |
| `bucks_support_activated_at` | ISO timestamp |
| `bucks_support_message_origin` | `agent_greeting` for A; `automated_visitor` for B |

The app writes these values automatically; agents do not configure them per shop. These are latest-activation context, not a historical ledger. Preserve each activation in PostHog and Crisp profile events, including its ID and timestamp. Where available, retain the Crisp conversation/session reference in the activation record to join weekly reviews. Do not include full chat text in analytics.

Tagging and custom-data updates are best-effort and must not interrupt the merchant action. Do not configure message automations against these labels or fields: the six dedicated milestone events are the trigger inputs. If a conversation unexpectedly has both variant segments, flag it for attribution review rather than silently merging groups or deleting unrelated tags.

### Agent review using this context

1. From PostHog, select unique activated shops whose full 7-day or 14-day observation window has elapsed, split by variant and enrollment cohort.
2. Locate their Crisp conversations using stored references and the experiment segment/context. Include shops with no reply or no conversation in the denominator; inbox-only sampling would omit non-conversions.
3. Review the relevant time window against the agreed meaningful-conversation rubric. B's automatic opening message is not genuine merchant participation.
4. Record reviewer, verdict (`meaningful`, `not_meaningful`, or `unclear`), evidence note, feature/activation ID, and observation window in the review ledger. Do not silently classify `unclear` as converted; resolve or report it separately before final comparison.
5. Compare unique shops with a confirmed meaningful conversation against all activated shops per variant. Report native milestone-trigger counts separately.

### Same-tab reply detection

Use a single app-level listener for Crisp `message:sent`, registered before delivery and coordinated with any existing listeners. Do not install one handler per feature component or remove another feature's listener indiscriminately.

Proposed tab state: experiment, variant, shop/installation scope, feature, activation ID/time, pending automatic-send correlation, and reply-recorded status. Persist only minimal attribution state in session storage. Deduplicate outcome handling by activation and, where available, Crisp message identifier.

**Implementation gate:** verify whether `message:sent` also fires for SDK-submitted visitor messages and what stable message fields can correlate the automatic send. Do not use "ignore the next message" or text equality alone: either can misclassify a merchant sending concurrently. If no reliable distinction exists, automatic-send confirmation and comparable human-reply measurement need a revised design before rollout.

The first actual merchant message after activation is a proxy for engagement, not proof of relevance to that feature. Historical messages, agent messages, profile events, and B automatic messages must not count. An already-active conversation can produce unrelated replies; this is an attribution limitation to report.

### Failures and repeat behavior

- Analytics failure must not block feature actions or chat delivery. Keep original event/claim behavior unless separately approved.
- No automatic replay of an ambiguously sent visitor message: duplicate support requests are worse than an undercount. Browser persistence and Crisp are not a single transaction.
- Existing completion retries must not replay activation or message delivery.
- App-controlled A greetings and B visitor sends retain app eligibility limits. Native milestone triggers remain subject to Crisp visitor suppression; their counts may be lower than actual milestones.
- Do not clear Crisp visitor identity, trigger history, or installation claims to balance exposure.
- A browser listener may observe messages from another tab sharing the same Crisp session; verify locality. If origin cannot be established, do not claim exact same-tab attribution.

### Phases and validation

1. **SDK proof and measurement contract:** validate send callbacks, automated-send correlation, callback coexistence, cross-tab behavior, and event persistence. Resolve the gates below before production traffic.
2. **Implementation:** add deterministic assignment and variant adapters, then persist/deduplicate activation and outcome attribution. Update Sidekick explanations, citations, and tests with behavior changes.
3. **Controlled validation:** exercise each feature and variant with eligible test shops; check PostHog and Crisp profiles, and support-operator interpretation of B.
4. **Rollout:** use an internal experiment enable/disable control, not a merchant setting. Disabling the experiment restores current A delivery without resetting claims or reassigning the same experiment population. Keep the assignment algorithm fixed.

Required automated coverage: stable shop assignment across pages; independent experiment key; A emits only agent-trigger activation; B sends only the visitor message; automatic sends excluded from human replies; first-reply deduplication; profile events never count as replies; navigation/reload; wrong shop/installation; concurrency; readiness failure; analytics failure; impersonation; and retained claim/timing behavior.

Manual evidence must establish observable send confirmation, exact profile/PostHog event properties, no duplicate greeting in B, and whether closed/reopened or cross-tab conversations affect attribution. No implementation or live validation has been performed for this RFC.

## 6. Open Questions and Launch Gates

1. **Engineering:** Can the live Crisp send callback reliably distinguish B's SDK-generated message from a human send, including concurrent activity and other tabs? Validate with captured callback payloads before selecting an attribution implementation.
2. **Product/analytics:** What baseline human-reply rate, minimum meaningful uplift, sample size, experiment duration, and support-workload ceiling should govern the decision? Select these before launch; do not copy the onboarding document's example numbers or stop on an early apparent winner.
3. **Product/support:** Confirm automatic visitor-message wording, including the cart grammar correction, and acceptance of conversations created without a merchant Send action. Does the celebratory cart message give operators enough context to respond appropriately?
4. **Engineering/Crisp support:** Can six silent milestone triggers count in native Trigger Analytics without message/opening actions? Confirm before implementation. If unsupported, return the reporting limitation for a decision rather than silently replacing native analytics with profile-event counts.

## References

- `docs/AB_TESTING_IMPLEMENTATION.md` - historical assignment design; verify implementation claims before reuse.
- `docs/crisp-profile-events-and-triggers.md` - context/message events and reporting limitations.
- `utils/extras/chatTriggers.js`, `components/hooks/useChatTriggers.js` - current automatic-help coordinator and timing.
- `utils/extras/crispProfileEvent.js`, `utils/extras/crispChat.js` - current Crisp integration boundaries.
- `pages/api/v1/user/chatTriggers.js` - installation-scoped consumption.
- [Crisp Web SDK](https://docs.crisp.chat/guides/chatbox-sdks/web-sdk/dollar-crisp/) - visitor message sending, `message:sent`, and profile events.
- [Crisp Message Triggers](https://help.crisp.chat/en/article/how-to-use-message-triggers-and-quick-replies-in-hugo-ai-agent-pq0znd/) - trigger behavior and remembered visitor executions.
