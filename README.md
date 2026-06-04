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
| `bucks_plus` | **Plus Plan** | **$9.99/mo** | Monthly | ❌ | New/free users |
| `bucks_premium_pro` | **Pro** | **$19.99/mo** | Monthly | ✅ Full | All |
| `bucks_premium_pro_annual` | **Pro Annual** | **$95.90/yr** (default) / **$83.93/yr** (partner) | Annual | ✅ Full | All |

### Pricing Math (all confirmed)

| Value | Amount |
|---|---|
| Pro Monthly full price | $19.99/mo |
| Pro Monthly with coupon (BUCKS50) | $9.99/mo (50.03% off) |
| Pro Annual default | $95.90/yr (60.02% off $239.88) |
| Pro Annual partner price | $83.93/yr (65.00% off $239.88) |
| Pro Annual compare/strikethrough price | $239.88/yr ($19.99 × 12) |
| Pro Annual monthly equivalent | ~$7.99/mo (default) / ~$6.99/mo (partner) |
| Plus Plan Monthly | $9.99/mo (no coupon, no discount) |

> **Partner discount for Pro Annual:** A single DB plan ID `bucks_premium_pro_annual` is used for all users. When a partner-referred user upgrades to Pro Annual, `initSubscription.js` detects `partnerData` and overrides `price.amount` to $83.93 before calling Shopify. `discountPercent` is 60% for default, 65% for partner. The `acceptedPrice` column stores the actual amount paid.

---

## 2. Coupon Design

- **Code:** `BUCKS50` — stored in `process.env.PRO_COUPON_CODE` (fallback: `"BUCKS50"`)
- **Applies to:** Pro Monthly (`bucks_premium_pro`) **only**
- **Does NOT apply to:** Plus Plan Monthly (`bucks_plus`), Pro Annual
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
| **New / Non-legacy (any plan, incl. Pro)** | 2-row | Plus Plan $9.99 · Pro $19.99 · Pro Annual $95.90 | Free card (full width) |
| **Legacy $7.99 monthly (still active)** | 2-row | Legacy $7.99 · Pro $19.99 · Pro Annual $95.90 | Free card (full width) |
| **Legacy annual (still active)** | 2-row | Pro $19.99 · Legacy Annual · Pro Annual $95.90 | Free card (full width) |
| **Former legacy (switched to any plan OR on free)** | Flat 3-col | Free · Pro $19.99 · Pro Annual $95.90 | *(none)* |

> **Key rule:** Non-legacy users (`isLegacyUser = false`) always see the Standard ($9.99) plan regardless of their current plan. Standard is only hidden for legacy users who have switched away from their legacy plan.

### 3b. Segment Detection Logic

```js
const isLegacyUser    = initialData?.userData?.isLegacyUser === true; // from DB, in initialDataSelectors
const isLegacyMonthly = currentPlan === "bucks_premium";
const isLegacyAnnual  = ["bucks_premium_60", "bucks_premium_65", "bucks_premium_70"].includes(currentPlan);
const isFormerLegacy  = isLegacyUser && !isLegacyMonthly && !isLegacyAnnual; // legacy user who switched away

const useTwoRowLayout = !isFormerLegacy; // flat layout only for former legacy users

const topRowIds = useMemo(() => {
  if (isLegacyMonthly) return ["legacy-monthly", "pro-monthly", "pro-annual"];
  if (isLegacyAnnual)  return ["pro-monthly", "legacy-annual", "pro-annual"];
  return ["plus", "pro-monthly", "pro-annual"]; // always for non-legacy (any current plan)
}, [isLegacyMonthly, isLegacyAnnual]);

const flatRowIds = ["free", "pro-monthly", "pro-annual"]; // former legacy: no plus plan
```

> **`isLegacyUser` must be in `initialDataSelectors`** (`utils/constants.js`) so SSR passes it to the pricing page.

### 3b-ii. Rendering Conditional

The two values above drive a single conditional in `pages/pricing/index.jsx`:

- **`useTwoRowLayout = true`** → renders `topRowIds` as a 3-column grid + Free card below as a separate full-width row
- **`useTwoRowLayout = false`** → renders `flatRowIds` as a single 3-column grid (Free is position 1 in the row, no separate row below)

The grid column count always stays at 3. No blank slots — every layout has exactly 3 cards in the grid row.

### 3c. Card ID → planDetails Mapping

| Card ID | plan_name (DB) | Shown When |
|---|---|---|
| `"free"` | `bucks_free` | Always (bottom row or flat position 1) |
| `"plus"` | `bucks_plus` | Non-legacy users only |
| `"pro-monthly"` | `bucks_premium_pro` | Always |
| `"pro-annual"` | `bucks_premium_pro_annual` | Always (new Pro Annual) |
| `"legacy-monthly"` | `bucks_premium` | Legacy monthly segment only |
| `"legacy-annual"` | `bucks_premium_60/65/70` | Legacy annual segment only |

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

**Former legacy user — switched to any plan (flat layout):**
```
┌──────────────────────────────────────────────────────────────┐
│  [ Free — $0   ]     [ Pro $19.99    ]   [ Pro Annual       ]│
│  [ Free plan  ]      [ Current plan ✓]   $95.90/yr          │
│                       Full analytics      Full analytics     │
│                       [Current plan  ]   [Upgrade & save]   │
└──────────────────────────────────────────────────────────────┘
```

### 3e. Standard Plan Display Name Rule

The Plus Plan card (`id: "plus"`, `plan_name: "bucks_plus"`) is **only visible to non-legacy users** — any user with `isLegacyUser = true` never sees it, whether they are currently on a legacy plan or have switched away.

| User Segment | Sees Standard card |
|---|---|
| New / Non-legacy (any current plan) | ✅ Always |
| Legacy monthly or annual (still active) | ❌ Not in `topRowIds` |
| Former legacy (switched to Pro, Free, or Plus) | ❌ Flat layout uses `flatRowIds` (no plus plan) |

Display name: always rendered as **"Plus Plan"** (`plan.id === "plus" ? "Plus Plan" : plan.name`).

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

**Discount badge (same pattern as legacy annual):**

| Partner Status | `discountPercent` | Badge Text |
|---|---|---|
| Non-partner | 60 | **"60% OFF"** |
| Partner | 65 | **"60% OFF"** + **"5% EXTRA DISCOUNT"** overlay |

The existing `PricingCard.jsx` annual badge logic uses `STANDARD_ANNUAL_DISCOUNT` (60) as the baseline and renders `(discountPercent - 60)% EXTRA DISCOUNT` when `partnerReferral && discountPercent > 60`. Pro Annual naturally fits this pattern.

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

**When user IS on `bucks_premium_pro_annual` (post-upgrade):** Show `acceptedPrice` from DB, calculate actual discount (60% default / 65% partner), button text → "Current plan".

---

## 5. `replacementBehavior` Rules

**Simplified rule:** `APPLY_IMMEDIATELY` for **every transition to any paid plan**, including from free.

| From Plan | To Plan | `replacementBehavior` |
|---|---|---|
| Free (`bucks_free`) | Plus / Pro Monthly / Pro Annual | **`"APPLY_IMMEDIATELY"`** |
| Legacy $7.99 | Plus / Pro Monthly / Pro Annual | **`"APPLY_IMMEDIATELY"`** |
| Legacy Annual | Plus / Pro Monthly / Pro Annual | **`"APPLY_IMMEDIATELY"`** |
| Plus | Pro Monthly / Pro Annual | **`"APPLY_IMMEDIATELY"`** |
| Pro Monthly | Plus / Pro Annual | **`"APPLY_IMMEDIATELY"`** |
| Pro Annual | Pro Monthly / Plus | **`"APPLY_IMMEDIATELY"`** |

**Detection logic:**
```js
const replacementBehavior = "APPLY_IMMEDIATELY"; // always — Shopify handles proration
```

> Shopify's `APPLY_IMMEDIATELY` works for all transitions including free-to-paid. The merchant is billed immediately for the prorated difference and any previous subscription is replaced.

---

## 6. Analytics Page Gate

### Access Matrix

| Plan | StatsCards | ChartsSection | PerformanceTables | SmartRecommendations |
|---|---|---|---|---|
| `bucks_free` | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| Legacy `bucks_premium` | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| Legacy annual plans | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
| `bucks_plus` | ✅ Visible | 🔒 Locked | 🔒 Locked | 🔒 Hidden |
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
const isAnnualPlan = id === "legacy-annual" || id === "pro-annual";
```

```js
// OLD — trial text only for legacy monthly
if (id === "legacy-monthly") { /* trial text logic */ }

// NEW — trial text also for new pro-monthly
if (id === "legacy-monthly" || id === "pro-monthly") { /* trial text logic */ }
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

## 8. Plan ID Safety: LEGACY_PLAN_IDS vs NEW_MONTHLY/ANNUAL_PLAN_IDS

**Critical architectural decision:** New plan IDs must NOT be added to `LEGACY_PLAN_IDS`.

**Why:** `isCurrentPlanCard` has a second condition:
```js
const planMatches = currentPlan === plan_text ||
    (plan_text === "bucks_premium" && LEGACY_PLAN_IDS.includes(currentPlan));
```
If `"bucks_premium_pro"` were in `LEGACY_PLAN_IDS`, a Pro user's DB plan would trigger the fallback and the **legacy $7.99 card** would incorrectly show as "Current plan."

**Solution:**
```js
// trialHelpers.js
export const LEGACY_PLAN_IDS = [           // Legacy only — do NOT add new IDs
  "bucks_premium",
  "bucks_premium_60",
  "bucks_premium_65",
  "bucks_premium_70",
];

export const NEW_MONTHLY_PLAN_IDS = [      // New monthly plans
  "bucks_plus",
  "bucks_premium_pro",
];

export const NEW_ANNUAL_PLAN_IDS = [       // New annual plans
  "bucks_premium_pro_annual",
];

export const NEW_PLAN_IDS = [...NEW_MONTHLY_PLAN_IDS, ...NEW_ANNUAL_PLAN_IDS]; // combined

export const isPremiumPlan = (currentPlan) =>
  LEGACY_PLAN_IDS.includes(currentPlan) || NEW_PLAN_IDS.includes(currentPlan);
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

> **Partner 30-day trial:** When a user comes through a partner referral (`partnerData` present), `partnerData.trialDays` overrides the default. If the partner config specifies 30 days, the Pro Annual card will show "Start your 30-day free trial" exactly as legacy annual plans do today. No new logic needed.

---

## 10. Backend API Payload Shapes

**Plus Plan Monthly (no coupon):**
```json
{
  "name": "Plus Plan",
  "plan_name": "bucks_plus",
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

**Pro Annual (partner) — what the frontend sends:**
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
→ Frontend **always sends `bucks_premium_pro_annual`** regardless of partner status. The backend detects `partnerData`, overrides `price.amount` to $83.93, and overrides `bucks_plan` to `bucks_premium_pro_annual_partner` before calling Shopify. The return URL will then carry `bucks_plan=bucks_premium_pro_annual_partner`.

---

## 11. PostHog Events

| Event Name | When | Key Props |
|---|---|---|
| `"bucks standard plan initiated"` | Standard card upgrade clicked | `myshopify_domain`, `bucks_plan: "bucks_premium_standard"`, `plan_type: "monthly"` |
| `"bucks pro plan initiated"` | Pro Monthly or Pro Annual upgrade clicked | `myshopify_domain`, `bucks_plan`, `plan_type` |
| `"bucks coupon applied"` | BUCKS50 applied on card | `myshopify_domain`, `coupon_code: "BUCKS50"`, `original_price: 19.99`, `final_price: 9.99` |
| `"bucks pro plan selected"` | Shopify billing callback confirms subscription | `myshopify_domain`, `bucks_plan`, `plan_type` |

---

## 12. `checkSubscriptionStatus.js` and `cancelSubscription.js` — `isLegacyUser` Flag Persistence

### Return URL (unchanged)

The return URL from `createAppSubscription` already includes:
```
...&bucks_plan=bucks_premium_pro&plan_type=monthly
...&bucks_plan=bucks_premium_pro_annual&plan_type=annual
...&bucks_plan=bucks_premium_pro_annual&plan_type=annual   ← partner pricing applied server-side, same plan ID
...&bucks_plan=bucks_plus&plan_type=monthly
```

### `isLegacyUser` write (new)

`checkSubscriptionStatus.js` reads `bucks_plan` and `plan_type` from query params and writes to DB. It also **stamps `isLegacyUser: true`** when the user's plan immediately before the upgrade was a legacy plan ID (`bucks_premium`, `bucks_premium_60`, `bucks_premium_65`, `bucks_premium_70`). The existing user row is fetched before the update, so the pre-upgrade plan is available for this check.

`cancelSubscription.js` applies the same check when downgrading to free: if the user's current plan before cancellation is a legacy plan ID, `isLegacyUser: true` is written alongside `bucks_plan: "bucks_free"`.

**The flag is sticky** — it is only ever written as `true`, never reset to `false`. Once a user is marked legacy, they remain legacy regardless of subsequent plan changes.

The partner override flows automatically because `finalDetails.bucks_plan` is set to the partner ID server-side before `createAppSubscription` is called.

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
