# Copy Coupon Button — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a one-click "Copy coupon" button to the Pro Monthly card that copies `BUCKS50` to clipboard and auto-fills the coupon input.

**Architecture:** A small inline button above the existing coupon input in `PricingCard.jsx`. Uses `navigator.clipboard.writeText()` with a 2-second "Copied!" inline feedback. Auto-fills the input field so the user clicks Copy → Apply with no typing.

**Tech Stack:** React, Polaris components (Button, InlineStack, Text), `navigator.clipboard` API

---

## File Map

| File | Responsibility | Action |
|---|---|---|
| `components/pricing/PricingCard.jsx` | Renders coupon input + copy button | Modify |
| `docs/superpowers/specs/2026-06-03-new-pricing-plans-design.md` | Spec already updated | No change |

---

### Task 1: Add `copied` state and `handleCopyCoupon` handler

**Files:**
- Modify: `components/pricing/PricingCard.jsx:87-91` (existing coupon state block)

- [ ] **Step 1: Add `copied` state alongside existing coupon states**

After the existing coupon state declarations (around line 90), add:

```js
const [copied, setCopied] = React.useState(false);
```

- [ ] **Step 2: Add `handleCopyCoupon` function after `handleApplyCoupon`**

Insert immediately after the `handleApplyCoupon` function body:

```js
const handleCopyCoupon = async () => {
  const COUPON_CODE = "BUCKS50";
  try {
    await navigator.clipboard.writeText(COUPON_CODE);
    setCopied(true);
    setCouponInput(COUPON_CODE);
    setTimeout(() => setCopied(false), 2000);
  } catch {
    // Silently ignore clipboard failures (e.g., insecure context)
  }
};
```

---

### Task 2: Render "Copy coupon" button in the default state

**Files:**
- Modify: `components/pricing/PricingCard.jsx:730-771` (existing coupon UI block)

- [ ] **Step 3: Add copy coupon button above the input field**

In the `{couponEnabled && (...)}` block, inside the default state (where `!couponDisplayPrice`), add an `InlineStack` above the input row that shows the coupon hint + copy button:

```jsx
{!couponDisplayPrice && (
  <div style={{ marginBottom: "8px" }}>
    <InlineStack gap="200" align="start" blockAlign="center">
      <Text as="span" variant="bodySm" tone="subdued">
        🎟️ Use code BUCKS50 for 50% off
      </Text>
      <Button
        size="micro"
        variant="plain"
        onClick={handleCopyCoupon}
        disabled={copied}
      >
        {copied ? "Copied!" : "Copy coupon"}
      </Button>
    </InlineStack>
  </div>
)}
```

> Place this block directly above the existing input/Apply row within the `!couponDisplayPrice` branch.

---

### Task 3: Verify the existing price-update behaviour

**Files:**
- No file changes needed — existing code already handles this

- [ ] **Step 4: Confirm `handleApplyCoupon` sets `couponDisplayPrice = 9.99`**

Verify the existing code at `components/pricing/PricingCard.jsx:248-265` already contains:

```js
const handleApplyCoupon = () => {
  const code = couponInput.trim().toUpperCase();
  if (!code) {
    setCouponError("Please enter a coupon code.");
    return;
  }
  setAppliedCoupon(code);
  setCouponDisplayPrice(9.99);  // ← price update already here
  setCouponError("");
  // ...PostHog tracking
};
```

And the applied-state render block (around line 732-747) already shows:
- Strikethrough `$19.99/mo`
- Green `$9.99/mo`
- "Remove" button to reset

If any of the above is missing, add it now.

---

### Task 4: Manual test

- [ ] **Step 5: Start dev server**

```bash
npm run dev
```

- [ ] **Step 6: Manual verification**

1. Log in as a non-legacy user
2. Navigate to Pricing page
3. Verify Pro Monthly card shows:
   - "🎟️ Use code BUCKS50 for 50% off [Copy coupon]" above the input
4. Click "Copy coupon"
   - Button text changes to "Copied!" for 2 seconds then reverts
   - Input field auto-fills with `BUCKS50`
5. Click "Apply"
   - Price display updates: ~~$19.99~~ → $9.99 in green
   - Subtitle updates to "Then $9.99/month with BUCKS50"
6. Click "Remove" — returns to default state

---

### Task 5: Commit

- [ ] **Step 7: Commit**

```bash
git add components/pricing/PricingCard.jsx
git commit -m "feat(pricing): add copy-coupon button to Pro Monthly card

- One-click copy of BUCKS50 to clipboard via navigator.clipboard
- Auto-fills coupon input after copy for zero-friction apply
- 2-second 'Copied!' inline feedback on the button
- Existing price update ($19.99 → $9.99) confirmed working"
```
