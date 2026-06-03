# New Pricing Plans — Design Spec

**Created:** June 3, 2026  
**Status:** Confirmed — Ready for Implementation  
**RFC:** `docs/rfc-new-pricing-plans.md`  
**Implementation Plan:** `docs/superpowers/plans/2026-06-03-new-pricing-plans.md`

> ⚠️ **Note:** `docs/new-pricing-plans.md` contains stale values ($14.99, $95.88, BUCKS33, "STANDARD" replacementBehavior, flat grid layout). This spec supersedes it with confirmed final values.

---

## 1. Plan Catalogue (Confirmed Final)

| DB ID | Display Name | Price | Interval | Analytics | Audience |
|---|---|---|---|---|---|
| `bucks_free` | Free | $0 | — | ❌ | All |
| `bucks_premium` | Legacy Plus | $7.99/mo | Monthly | ❌ | Legacy subscribers only |
| `bucks_premium_60/65/70` | Legacy Annual | varies/yr | Annual | ❌ | Legacy subscribers only |
| `bucks_premium_standard` | **Plus Plan** *(or "Standard")* | **$9.99/mo** | Monthly | ❌ | New/free users |
| `bucks_premium_pro` | **Pro** | **$19.99/mo** | Monthly | ✅ Full | All |
| `bucks_premium_pro_annual` | **Pro Annual** | **$95.90/yr** | Annual | ✅ Full | All |

### Pricing Math (all confirmed)

| Value | Amount |
|---|---|
| Pro Monthly full price | $19.99/mo |
| Pro Monthly with coupon (BUCKS50) | $9.99/mo (50.03% off) |
| Pro Annual default | $95.90/yr (39.98% off $239.88) |
| Pro Annual partner price | $83.93/yr (34.98% off $239.88) |
| Pro Annual compare/strikethrough price | $239.88/yr ($19.99 × 12) |
| Pro Annual monthly equivalent | ~$7.99/mo |
| Standard Monthly | $9.99/mo (no coupon, no discount) |

---

## 2. Coupon Design

- **Code:** `BUCKS50` — stored in `process.env.PRO_COUPON_CODE` (fallback: `"BUCKS50"`)
- **Applies to:** Pro Monthly (`bucks_premium_pro`) **only**
- **Does NOT apply to:** Standard Monthly, Pro Annual
- **Discount:** $19.99 → $9.99/mo (50.03% off)
- **Validation:** Backend only. Frontend shows instant price preview on apply; wrong codes are silently ignored server-side (no price change)
- **UI location:** Input field with Apply button, rendered **on the Pro Monthly card only** (`couponEnabled: true` flag in planDetails)

### Coupon Input States

```
── State 1: Default (no coupon entered) ──────────────────────────
  [         Enter coupon code          ] [ Apply ]
  Then $19.99/month · Cancel anytime

── State 2: Applied ──────────────────────────────────────────────
  ~~$19.99/mo~~   $9.99/mo ✓   [ Remove ]
  Then $9.99/month with BUCKS50 · Cancel anytime

── State 3: Empty submit ─────────────────────────────────────────
  [         Enter coupon code          ] [ Apply ]
  Please enter a coupon code.   ← red error text

── State 4: Wrong code (frontend-only note) ──────────────────────
  Client accepts any non-empty code (shows $9.99 preview).
  Backend validates — if invalid, Shopify charges $19.99.
  Design decision: No client-side code rejection.
```

---

## 3. Pricing Page Layout

### 3a. Layout Rule

| User Segment | Layout Mode | Top Row (3 cols) | Bottom |
|---|---|---|---|
| **New / Free / Standard** | 2-row | Plus Plan $9.99 · Pro $19.99 · Pro Annual $95.90 | Free card (full width) |
| **Legacy $7.99 monthly (active)** | 2-row | Legacy $7.99 · Pro $19.99 · Pro Annual $95.90 | Free card (full width) |
| **Legacy annual (active)** | 2-row | Pro $19.99 · Legacy Annual · Pro Annual $95.90 | Free card (full width) |
| **Pro Monthly user (post-upgrade)** | Flat 3-col | Free · Pro $19.99 (Active) · Pro Annual $95.90 | *(none)* |
| **Pro Annual user (post-upgrade)** | Flat 3-col | Free · Pro $19.99 · Pro Annual $95.90 (Active) | *(none)* |

### 3b. Segment Detection Logic

```js
const isLegacyMonthly = currentPlan === "bucks_premium";
const isLegacyAnnual  = ["bucks_premium_60", "bucks_premium_65", "bucks_premium_70"].includes(currentPlan);
const isOnProPlan     = ["bucks_premium_pro", "bucks_premium_pro_annual"].includes(currentPlan);

const useTwoRowLayout = !isOnProPlan;  // all users except Pro Monthly/Annual post-upgrade
```

### 3c. Card ID → planDetails Mapping

| Card ID | plan_name (DB) | Shown When |
|---|---|---|
| `"free"` | `bucks_free` | Always (bottom row or flat position 1) |
| `"standard"` | `bucks_premium_standard` | Free/new/standard segment |
| `"pro-monthly"` | `bucks_premium_pro` | Always |
| `"new-pro-annual"` | `bucks_premium_pro_annual` | Always |
| `"pro"` | `bucks_premium` | Legacy monthly segment only |
| `"pro-annual"` | `bucks_premium` (legacy) | Legacy annual segment only |

### 3d. ASCII Wireframes

**New / Free / Standard users:**
```
┌──────────────────────────────────────────────────────────────┐
│  [ Plus Plan $9.99 ]  [ Pro $19.99     ]  [ Pro Annual      ]│
│  Billed monthly       Coupon: BUCKS50     $95.90/yr          │
│  No analytics         Full analytics      Full analytics     │
│  [Start 7-day trial]  [Start 7-day trial] [Upgrade & save]  │
└──────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────┐
│                    [ Free — $0 ]                             │
│                    [ Current plan ] (if on free)             │
└──────────────────────────────────────────────────────────────┘
```

**Legacy $7.99 monthly user (active):**
```
┌──────────────────────────────────────────────────────────────┐
│  [ Legacy $7.99   ]   [ Pro $19.99    ]   [ Pro Annual      ]│
│  [Current plan  ✓]   Coupon: BUCKS50     $95.90/yr          │
│                       Full analytics      Full analytics     │
│  [Current plan  ]    [Start trial    ]   [Upgrade & save]   │
└──────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────┐
│                    [ Free — $0 ]                             │
│                    [ Free plan ]                             │
└──────────────────────────────────────────────────────────────┘
```

**Legacy annual user (active):**
```
┌──────────────────────────────────────────────────────────────┐
│  [ Pro $19.99     ]  [ Legacy Annual  ]  [ Pro Annual       ]│
│  Coupon: BUCKS50     [Current plan  ✓]   $95.90/yr          │
│  Full analytics                           Full analytics     │
│  [Start trial    ]   [Current plan  ]    [Upgrade & save]   │
└──────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────┐
│                    [ Free — $0 ]                             │
│                    [ Free plan ]                             │
└──────────────────────────────────────────────────────────────┘
```

**Pro Monthly user (post-upgrade, flat layout):**
```
┌──────────────────────────────────────────────────────────────┐
│  [ Free — $0   ]     [ Pro $19.99    ]   [ Pro Annual       ]│
│  [ Free plan  ]      [ Current plan ✓]   $95.90/yr          │
│                       Full analytics      Full analytics     │
│                       [Current plan  ]   [Upgrade & save]   │
└──────────────────────────────────────────────────────────────┘
```

### 3e. Standard Plan Display Name Rule

| User Segment | `id: "standard"` renders as |
|---|---|
| Free / New (no legacy plan) | **"Plus Plan"** |
| Legacy $7.99 monthly (active) | **"Standard"** |
| Legacy annual (active) | **"Standard"** |

Logic: `plan.id === "standard" ? (hasActiveLegacy ? "Standard" : "Plus Plan") : plan.name`

---

## 4. Pro Annual Card Pricing Display

The Pro Annual card uses the existing annual card rendering in `PricingCard.jsx`. Values passed from `pricing/index.jsx`:

| Value | Non-Partner | Partner |
|---|---|---|
| `annualTotal` | $95.90 | $83.93 |
| `displayAmount` (monthly equiv) | $7.99/mo | ~$6.99/mo |
| `displayComparePrice` (monthly equiv) | $19.99/mo | $19.99/mo |
| `discountPercent` | 60% | 65% |
| `savingsText` | "You save $143.98 per year" | "You save $155.95 per year" |
| `dynamicButtonText` | "Upgrade and save $143.98 now" | "Upgrade and save $155.95 now" |

**Card visual (top section):**
```
Pro Annual                          [BEST VALUE]
Lock in analytics with 60% annual saving

~~$19.99/mo~~       60% OFF
$7.99
    / month

┌─────────────────────────────────┐
│ Billed annually         HUGE DEAL│
│ $95.90                   / year  │
│ ─────────────────────────────── │
│ ✦ You save $143.98 per year     │
│   vs regular price of $239.88/yr│
└─────────────────────────────────┘
```

**When user IS on `bucks_premium_pro_annual` (post-upgrade):** Show `acceptedPrice` from DB, calculate actual discount, button text → "Current plan".

---

## 5. `replacementBehavior` Rules

| From Plan | From Interval | To Plan | To Interval | `replacementBehavior` |
|---|---|---|---|---|
| Free / Standard | — / monthly | Any | Any | `undefined` |
| Legacy $7.99 | Monthly | Standard / Pro Monthly | Monthly | **`"APPLY_IMMEDIATELY"`** |
| Legacy $7.99 | Monthly | Pro Annual | Annual | `undefined` (cross-interval) |
| Legacy Annual | Annual | Standard / Pro Monthly | Monthly | `undefined` (cross-interval) |
| Legacy Annual | Annual | Pro Annual | Annual | **`"APPLY_IMMEDIATELY"`** |
| Pro Monthly | Monthly | Standard | Monthly | **`"APPLY_IMMEDIATELY"`** |
| Pro Monthly | Monthly | Pro Annual | Annual | `undefined` (cross-interval) |
| Pro Annual | Annual | Pro Monthly | Monthly | `undefined` (cross-interval) |

**Detection logic:**
```js
const isOnActivePaidPlan = (PREMIUM_PLAN_IDS.includes(currentPlan) || NEW_PLAN_IDS.includes(currentPlan)) && !!subscriptionId;
const isSameInterval = !!currentInterval && currentInterval === newInterval;
const replacementBehavior = isOnActivePaidPlan && isSameInterval ? "APPLY_IMMEDIATELY" : undefined;
```

---

## 6. Analytics Page Gate

### Access Matrix

| Plan | StatsCards | ChartsSection | PerformanceTables | SmartRecommendations |
|---|---|---|---|---|
| `bucks_free` | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| Legacy `bucks_premium` | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| Legacy annual plans | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| `bucks_premium_standard` | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| `bucks_premium_pro` | ✅ Visible | ✅ Visible | ✅ Visible | ✅ Visible |
| `bucks_premium_pro_annual` | ✅ Visible | ✅ Visible | ✅ Visible | ✅ Visible |

### Locked Section UI (AnalyticsUpgradeLock component)

```
┌──────────────────────────────────────────────┐
│  [blurred/greyed placeholder — height 200px] │
│                                              │
│         ┌───────────────────────┐            │
│         │  Unlock Full Analytics│            │
│         │  Upgrade to Pro to    │            │
│         │  access charts,       │            │
│         │  tables, and AI recs. │            │
│         │  [  Upgrade to Pro  ] │            │
│         └───────────────────────┘            │
└──────────────────────────────────────────────┘
```

- Blur: `filter: blur(2px)` on placeholder background div
- Overlay: `rgba(255,255,255,0.75)` positioned absolutely, centered card
- "Upgrade to Pro" button: `router.push('/pricing?shop={shop}')`
- SmartRecommendations: hidden entirely (`{hasFullAnalytics && <SmartRecommendations />}`) — no lock overlay

---

## 7. PricingCard Component Changes

### New props added

| Prop | Type | Purpose |
|---|---|---|
| `couponEnabled` | `boolean` | Renders coupon input on this card (Pro Monthly only) |

### Updated conditionals

```js
// OLD — only legacy IDs
const isAnnualPlan = id === "pro-annual";

// NEW — includes new Pro Annual
const isAnnualPlan = id === "pro-annual" || id === "new-pro-annual";
```

```js
// OLD — trial text only for legacy "pro"
if (id === "pro") { /* trial text logic */ }

// NEW — trial text also for new pro-monthly
if (id === "pro" || id === "pro-monthly") { /* trial text logic */ }
```

### Coupon input — local state

```js
const [couponInput, setCouponInput] = React.useState("");
const [appliedCoupon, setAppliedCoupon] = React.useState(null);
const [couponDisplayPrice, setCouponDisplayPrice] = React.useState(null); // 9.99 when applied
const [couponError, setCouponError] = React.useState("");
```

### Upgrade button onClick

```js
// OLD
onClick={() => id === "free" ? downgradeHandler() : upgradePlanHandler(plan)}

// NEW — includes coupon in payload when applied
onClick={() => id === "free"
  ? downgradeHandler()
  : upgradePlanHandler({ ...plan, couponCode: appliedCoupon || undefined })}
```

---

## 8. Plan ID Safety: PREMIUM_PLAN_IDS vs NEW_PLAN_IDS

**Critical architectural decision:** New plan IDs must NOT be added to `PREMIUM_PLAN_IDS`.

**Why:** `isCurrentPlanCard` has a second condition:
```js
const planMatches = currentPlan === plan_text ||
    (plan_text === "bucks_premium" && PREMIUM_PLAN_IDS.includes(currentPlan));
```
If `"bucks_premium_pro"` were in `PREMIUM_PLAN_IDS`, a Pro user's DB plan would trigger the fallback and the **legacy $7.99 card** would incorrectly show as "Current plan."

**Solution:**
```js
// trialHelpers.js
export const PREMIUM_PLAN_IDS = [  // LEGACY ONLY — do NOT add new IDs
  "bucks_premium",
  "bucks_premium_60",
  "bucks_premium_65",
  "bucks_premium_70",
];

export const NEW_PLAN_IDS = [       // NEW — separate from legacy
  "bucks_premium_standard",
  "bucks_premium_pro",
  "bucks_premium_pro_annual",
];

export const isPremiumPlan = (currentPlan) =>
  PREMIUM_PLAN_IDS.includes(currentPlan) || NEW_PLAN_IDS.includes(currentPlan);
```

New plan cards use **exact match** (`currentPlan === plan_text`) since their `plan_name` in planDetails is unique.

---

## 9. Trial Days Logic (Unchanged)

| User State | Trial Days |
|---|---|
| Never subscribed (no `subscription_id`) | `partnerData.trialDays` or `DEFAULT_TRIAL_DAYS` (7) |
| On premium with active `trialEndDate` | Remaining days from `trialEndDate` |
| On free with preserved `trialLeft` | `trialLeft` value |
| Previously used trial, now free | 0 |

Applies identically to Standard, Pro Monthly, and Pro Annual.

---

## 10. Backend API Payload Shapes

**Standard Monthly (no coupon):**
```json
{
  "name": "Plus Plan",
  "plan_name": "bucks_premium_standard",
  "plan_type": "monthly",
  "price": { "amount": 9.99, "currencyCode": "USD" },
  "interval": "EVERY_30_DAYS",
  "trialDays": 7
}
```

**Pro Monthly with coupon:**
```json
{
  "name": "Pro",
  "plan_name": "bucks_premium_pro",
  "plan_type": "monthly",
  "price": { "amount": 19.99, "currencyCode": "USD" },
  "interval": "EVERY_30_DAYS",
  "trialDays": 7,
  "couponCode": "BUCKS50"
}
```
→ Backend reads `process.env.PRO_COUPON_CODE`, validates `couponCode === validCoupon`, overrides `price.amount = 9.99`.

**Pro Annual (non-partner):**
```json
{
  "name": "Pro Annual",
  "plan_name": "bucks_premium_pro_annual",
  "plan_type": "annual",
  "price": { "amount": 95.90, "currencyCode": "USD" },
  "interval": "ANNUAL",
  "trialDays": 7
}
```
→ Backend ignores frontend `price.amount` and sets it: `partnerData ? 83.93 : 95.90`.

---

## 11. PostHog Events

| Event Name | When | Key Props |
|---|---|---|
| `"bucks standard plan initiated"` | Standard card upgrade clicked | `myshopify_domain`, `bucks_plan: "bucks_premium_standard"`, `plan_type: "monthly"` |
| `"bucks pro plan initiated"` | Pro Monthly or Pro Annual upgrade clicked | `myshopify_domain`, `bucks_plan`, `plan_type` |
| `"bucks coupon applied"` | BUCKS50 applied on card | `myshopify_domain`, `coupon_code: "BUCKS50"`, `original_price: 19.99`, `final_price: 9.99` |
| `"bucks pro plan selected"` | Shopify billing callback confirms subscription | `myshopify_domain`, `bucks_plan`, `plan_type` |

---

## 12. `checkSubscriptionStatus.js` — No Changes

The return URL from `createAppSubscription` already includes:
```
...&bucks_plan=bucks_premium_pro&plan_type=monthly
...&bucks_plan=bucks_premium_pro_annual&plan_type=annual
...&bucks_plan=bucks_premium_standard&plan_type=monthly
```
`checkSubscriptionStatus.js` reads `bucks_plan` and `plan_type` from query params and writes to DB — no changes needed.

---

## 13. Stale Document Notice

`docs/new-pricing-plans.md` contains the following confirmed errors and should be treated as **deprecated**. This spec supersedes it:

| Field | Stale Value | Correct Value |
|---|---|---|
| Pro Monthly price | $14.99 | **$19.99** |
| Pro Annual price | $95.88 | **$95.90** |
| Pro Annual compare price | $119.88 | **$239.88** |
| Coupon code | BUCKS33 | **BUCKS50** |
| Coupon source | hardcoded | **`process.env.PRO_COUPON_CODE`** |
| replacementBehavior value | `"STANDARD"` | **`"APPLY_IMMEDIATELY"`** |
| Layout: free/new users | 4-card flat grid | **3 premium top + free below** |
| Layout: legacy users | 5-card flat grid | **3-card top row + free below** |
| `PREMIUM_PLAN_IDS` | add new IDs | **keep legacy-only; use NEW_PLAN_IDS** |
| Analytics lock UX | "TBD" | **blur overlay + upgrade CTA card** |
| planDetails "pro" ID | conflicts with legacy | **use "pro-monthly"** |

---

*Spec complete. Awaiting user review before implementation proceeds.*


