# RFC: BUCKS Integration With Shopify Sidekick

- **Status:** Draft
- **Type:** Feature RFC
- **Audience:** Merchants using BUCKS in Shopify Admin
- **Proposed reviewers:** BUCKS product, engineering, and support teams
- **Scope:** Core proposal for alignment; a detailed technical RFC follows approval.

## 1. Problem

Merchants need help setting up BUCKS, understanding currency behavior, and troubleshooting their stores. Answers currently require navigating app settings, reading help articles, or contacting support.

Sidekick should make basic guidance and store-specific configuration easier to understand without leaving the merchant's current workflow.

## 2. Why This Matters

- Make setup and common troubleshooting easier.
- Reduce searching across settings and documentation.
- Explain the differences between displayed currency, market currency, and checkout currency.
- Help merchants understand analytics without confusing currency selections with purchases.
- Direct issues requiring investigation to BUCKS support with useful context.

These are intended outcomes. Support-ticket reduction should be measured after release rather than assumed.

## 3. Proposed Solution

Provide a **read-only Sidekick integration** that answers basic BUCKS questions using approved product guidance and authenticated store configuration, then links to the relevant app page or help resource.

| Answer type | Purpose | Example |
| --- | --- | --- |
| Product guidance | Explain verified functionality and documented steps | "How do I add a currency?" |
| Store-specific answers | Explain the merchant's saved configuration | "Which currencies have I enabled?" |
| Guided troubleshooting | Identify configuration-based causes and suggest basic checks | "Why might my switcher be hidden on mobile?" |
| Navigation and support | Link to settings, help, or support | "Where can I get help with this issue?" |

**Recommended first release:** approved product guidance, saved configuration, and basic troubleshooting. Analytics definitions can be included once verified; store-specific analytics reporting is a later phase pending validation of metric definitions, access controls, and response speed.

Compared with a general FAQ alone, this approach provides relevant store context. Compared with a full data-and-actions integration, it keeps the initial scope smaller and leaves changes in the existing app UI. No new BUCKS settings screen is proposed.

### Core Merchant Questions

This is a question inventory, not a claim that every answer or data source is already available. Each launch topic requires a verified source.

| Area | Questions |
| --- | --- |
| Setup | How do I enable BUCKS on my theme? Why isn't the currency switcher showing? Do I need to remove my old currency converter? Do I need to enable BUCKS again after changing or publishing a theme? |
| Currencies | How do I add or remove currencies? Can I change the default currency? Can customers choose their own currency? Which currencies are currently enabled? |
| Automatic conversion | Can BUCKS detect the visitor's country? Why am I seeing the wrong currency? Will it remember a customer's selection? Which takes priority: automatic detection, the default currency, or a saved selection? |
| Prices and exchange rates | Where do exchange rates come from? How often are they updated? Can I use a manual rate or round converted prices? Why does the converted price look incorrect? Does BUCKS change product prices or only their display? |
| Checkout | Will customers pay in the currency they select? Why does the currency change at checkout? How does BUCKS work with Shopify Markets? |
| Appearance | Can I change the switcher's position, colors, or flags? Can I hide it on mobile? What are my current visibility settings? |
| Compatibility and troubleshooting | Does BUCKS work with my theme? Why aren't cart prices converting? Does it work with quick-view popups or subscription apps? Will it slow down my store? |
| Plans and support | What is included in the free plan? How do I cancel my subscription? How do I contact support? What information should I share for an investigation? |
| Markets | Does BUCKS use currencies configured in my markets? Does changing currency change the customer's market? Does BUCKS respect market-specific prices? What happens if a visitor's country is not in an active market? Do I need to update BUCKS after changing market settings? |
| Analytics | How are visits tracked and stored? What counts as a visit? Can I see visits by country or currency? How many shoppers clicked the switcher or changed currency? Why do numbers differ from Shopify Analytics? How long is data stored? Why is there no data yet, and when was it last updated? |
| Testing | How can I test another currency or country without affecting customers? Why does an incognito window or another device show a different currency? |

### Support Escalation - Core Requirement

> **Sidekick provides basic guidance and explains the merchant's BUCKS configuration. For further assistance, detailed investigation, or issues that cannot be resolved through basic checks, it must recommend contacting BUCKS support and provide an approved direct support link.**

Escalate when:

- The switcher or converted prices still do not work after basic checks.
- Theme, cart, quick-view, or third-party app compatibility requires investigation.
- Checkout, Shopify Markets, or analytics discrepancies cannot be explained using verified information.
- The merchant needs store-specific customization or more detailed assistance.

Briefly summarize the issue and checks already completed so the merchant can share them with support. Do not invent a diagnosis or imply that support has been contacted automatically.

**Example:**

> Your saved settings show that the switcher is enabled on mobile. If it still is not appearing, contact BUCKS support for a closer look at your theme. Share the affected page URL and a screenshot, along with the checks you have already tried.

The actual response must include the approved support link.

### Answer Boundaries

- Distinguish documented behavior, saved configuration, and verified live behavior.
- Do not treat a saved enabled status as proof that the widget works on the live theme.
- Do not guarantee checkout currency based on BUCKS display settings.
- Do not claim compatibility, performance, exchange-rate cadence, or retention policies without verified evidence.
- Keep plan answers factual; exclude upgrade nudges, promotions, and cross-selling.
- Identify missing, stale, or unavailable information rather than guessing.
- Do not expose credentials, unrelated personal data, or another merchant's information.
- No automatic setting changes, live storefront debugging, or automatic support-ticket creation in this release.

## 4. End-to-End Flow

```text
Merchant asks a BUCKS question in Sidekick
  -> Sidekick selects the relevant BUCKS data tool
  -> BUCKS returns approved guidance or authenticated store data
  -> Sidekick explains the answer and links to the relevant page
  -> If further investigation is needed, recommend BUCKS support
```

For example, if a merchant asks why the switcher is missing on mobile and saved mobile visibility is disabled, explain the setting and link to it. If visibility is enabled, suggest basic checks and escalate unresolved issues rather than claiming a live diagnosis.

Merchants make changes through the existing app. This release introduces no shopper-facing behavior changes.

## 5. Dependencies and Release Criteria

### Integration Prerequisites

- Support Shopify managed installation and token exchange; the inspected app currently uses legacy OAuth.
- Allow authenticated requests from Shopify's extension sandbox.
- Verify Shopify CLI 3.90.0 or later and the required Sidekick app configuration, including `extensions_summary`.
- Keep backend responses shop-scoped and enforce feature access server-side.
- Keep the headless Sidekick extension separate from the storefront widget build.

### Content Prerequisites

- Select approved BUCKS help and product sources for the launch question set.
- Verify rate provider/cadence, currency-selection precedence, Markets behavior, analytics definitions, and retention before answering those topics.
- Assign ownership for approving answers and keeping them current.
- Confirm the direct support link and relevant app/help destinations.

### Initial Acceptance Criteria

- Agreed launch questions have verified expected answers and sources.
- Configuration answers match the authenticated merchant's saved settings.
- Missing or stale information is explicitly identified.
- Unresolved issues and requests for detailed investigation lead to a clear support recommendation and working support link.
- No tool changes merchant settings or claims to contact support automatically.
- Links open the correct app page, help resource, or support destination.
- Responses stay within Shopify's 4,000-token limit and approximately one-second app-data response target.

Evaluate answer correctness, support-escalation behavior, and response latency before release. After launch, track tool failures and unanswered question categories, and measure support handoffs where observable.

### Phasing

1. **Core release:** product guidance, saved configuration, basic troubleshooting, and support handoff.
2. **Analytics expansion:** store-specific reporting after validating metric meaning, entitlements, freshness, and latency.

The detailed technical RFC will define tool schemas, endpoints, content delivery, caching, authentication migration, and rollout checks. This core RFC does not approve implementation.

## 6. Open Questions

1. Which BUCKS help center or support FAQ is the approved source for product guidance?
2. Should all listed topics receive general guidance at launch, with store-specific analytics deferred?
3. Who owns answer approval and updates when BUCKS behavior changes?
4. Which approved support destination should Sidekick link to?

## References

- [Shopify Sidekick app extensions](https://shopify.dev/docs/apps/build/sidekick)
- [Use extensions to surface app data](https://shopify.dev/docs/apps/build/sidekick/build-app-data)
