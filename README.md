# Currency Conversion Logic

## Overview

This document explains how the **Bucks Currency Converter** changes prices on a Shopify store. It covers the process from fetching exchange rates to updating the price displayed to the customer.

---

## 1. The Basic Concept (Beginner Friendly)

Imagine you are a customer from **Europe** visiting a store based in the **USA**.
1.  **Detection**: The app detects you are from Europe (using your IP address).
2.  **Rate Fetching**: The app knows 1 USD = 0.92 EUR (it fetches this rate daily).
3.  **Calculation**: It takes the product price ($100) and multiplies it by the rate (0.92).
4.  **Display**: It replaces "$100.00" with "€92.00" on your screen.

---

## 2. Technical Flow: Step-by-Step

### Step 1: Initialization & Rate Fetching
**File:** `extensions/bucks/blocks/app-embed.liquid` (and `initialize.js`)

When the storefront loads:
-   The app embed injects a global object `window.bucksCC`.
-   It attempts to use "Instant Loader" to fetch the core script (`sdk.min.js`) quickly.
-   **Rates**:
    -   If the store has a premium plan, it might use specific rules.
    -   Otherwise, it fetches rates from our central service or uses cached rates.
    -   The conversion formula is bound to `window.bucksCC.Currency.convert`.

### Step 2: Detecting Prices (`reconvert.js`)
**File:** `widgets/src/apps/widgets/buckscc/common/reconvert.js`

-   The app doesn't just run once. It needs to handle dynamic content (like drawer carts opening).
-   The `reconvert()` function is the trigger. It checks:
    -   Is the current currency different from the store's currency?
    -   Should we convert?
-   If yes, it calls `doPageConvert`.

### Step 3: Finding & Converting Elements (`converter.js`)
**File:** `widgets/src/apps/widgets/buckscc/convert/converter.js`

This is where the magic happens for a specific HTML element (like a price tag).

1.  **Read Value**: It looks for a `bucks-original` attribute. If not found, it tries to parse the text content of the element to find a number.
2.  **Calculate**:
    ```javascript
    // Simplified logic
    const rate = rates[targetCurrency];
    const newPrice = (originalPrice * rate);
    ```
3.  **Rounding**:
    -   If "Rounding" is enabled (e.g., round to .99), the `roundPrice` helper is called.
    -   *Example*: 15.23 becomes 15.99.
4.  **Formatting**:
    -   It formats the number according to the currency's standard (e.g., `,` for thousands, `.` for decimals).
    -   It applies the template (e.g., `{{amount}} EUR`).

### Step 4: Updating the DOM (`showUpdatedPrice.js`)
-   The app replaces the text content of the element with the new, formatted price.
-   It sets attributes like `bucks-currency="EUR"` to verify the conversion is done and avoid double-converting.

---

## 3. Key Files & Functions

| File/Function | Role |
| :--- | :--- |
| `initialize.js` | Sets up the specific `convert` math function (Window scope). |
| `reconvert.js` | The "Manager" that decides *when* to scan the page. |
| `converter.js` | The "Worker" that processes a *single* price element. |
| `roundPrice.js` | Applies psychological pricing rules (e.g., ending in .99). |
| `app-embed.liquid` | The entry point that loads our script into the theme. |

---

## 4. Frontend Integration

The app uses a **Theme App Extension** (`extensions/bucks`).
-   It does **not** edit theme files directly.
-   It injects a script that runs on the client-side (End-User's browser).
-   This means the conversion happens *after* the page initially loads (or concurrently), which is why "Instant Loader" is important to prevent a "flicker" of the old price.
