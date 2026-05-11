# Smart Recommendations — Technical Implementation

## Overview

The Suggestions system generates up to **3 actionable recommendations** per store based on analytics data. It has **two generation paths**: rule-based (manual) and AI-powered (Gemini). AI suggestions are preferred when available; manual suggestions serve as the fallback.

---

## Architecture

```
Frontend (bucks-converter-main)          Backend (bucks-analytics-backend)
─────────────────────────────────        ─────────────────────────────────
analytics/index.jsx                      
  │                                      
  │  GET /api/v1/analytics/proxy         
  │      ?path=suggestions               
  │      &startDate=...&endDate=...      
  │                                      
  └──► proxy.js ──► GET /api/analytics/suggestions?shop=...&startDate=...&endDate=...
                                           │
                                           ▼
                                     suggestionsController.ts → getSuggestions()
                                           │
                                     ┌─────┴──────────────────┐
                                     │  fetchSnapshotData()    │  (internal — gathers all data)
                                     │   ├─ Prisma: users,    │
                                     │   │  settings           │
                                     │   ├─ Shopify GraphQL:   │
                                     │   │  fetchAllMarkets()  │
                                     │   └─ ClickHouse:        │
                                     │      7 parallel queries │
                                     └────────────────────────-┘
                                           │
                                     ┌─────┴────────────────────┐
                                     │  generateSuggestions()    │  (manual / rule-based)
                                     │  generateAISuggestions()  │  (Gemini 2.0 Flash)
                                     └──────────────────────────┘
                                           │
                                     Response: { suggestions, aiSuggestions,
                                                 manualSuggestions, usingAI }
```

### Key Files

| Layer | File | Purpose |
|-------|------|---------|
| **Route** | `src/routes/analyticsRoutes.ts` | `GET /suggestions` → `getSuggestions` |
| **Controller** | `src/controllers/suggestionsController.ts` | Orchestrates data fetch → manual + AI generation |
| **Manual logic** | `src/utils/analytics/suggestionUtils.ts` | `generateSuggestions()`, `findCountriesNotInMarkets()`, `filterAISuggestions()` |
| **AI service** | `src/utils/analytics/aiSuggestionsService.ts` | `generateAISuggestions()` — calls Gemini, manages cache |
| **AI prompts** | `src/utils/analytics/aiPromptUtils.ts` | `buildSystemPrompt()`, `buildUserMessage()` — structures the prompt sent to Gemini |
| **Market utils** | `src/utils/analytics/marketUtils.ts` | `fetchAllMarkets()`, `findMarketContainingCountry()`, `getCountriesInMarket()` |
| **Constants** | `src/utils/analytics/constants.ts` | `ANALYTICS_THRESHOLDS` — all minimum thresholds |
| **Frontend proxy** | `pages/api/v1/analytics/proxy.js` | Authenticates session, forwards request to analytics backend |
| **Frontend page** | `pages/analytics/index.jsx` | Calls proxy in parallel with other analytics APIs |

---

## API

### `GET /api/analytics/suggestions`

**Query params:** `shop` (required), `startDate`, `endDate`

**Response:**
```json
{
  "success": true,
  "suggestions": [],       // Final list shown to user (AI if available, else manual)
  "aiSuggestions": [],      // AI-generated only
  "manualSuggestions": [],  // Rule-based only
  "aiFromCache": false,     // Whether AI result was served from DB cache
  "usingAI": true           // true if aiSuggestions were used
}
```

---

## Thresholds

Defined in `constants.ts`:

| Threshold | Value | Used For |
|-----------|-------|----------|
| `MIN_CONVERSIONS_THRESHOLD` | 1 | Minimum total currency switches before any currency suggestion triggers |
| `MIN_CURRENCY_SWITCHES` | 1 | Minimum switches for a specific currency to be suggested |
| `MIN_SALES_RATIO` | 1 | Minimum average order value (`totalSales / totalOrders`) for pricing/country suggestions |

---

## Suggestion Types

### 1. `CURRENCY_NOT_SELECTED` — Add Currency to Selected Currencies

**Trigger conditions (all must be true):**
- `totalConversions >= MIN_CONVERSIONS_THRESHOLD`
- A currency exists in `currencyConversions` that is **not** in the shop's `selectedCurrencies`
- That currency is **not** the shop's `defaultCurrency`
- That currency has `conversions >= MIN_CURRENCY_SWITCHES`
- Only the **top** (highest conversions) qualifying currency is suggested

**Data source:** `currency_switches` ClickHouse table → aggregated by currency

**Implementation:** `findCurrencyNotInSelectedList()` in `suggestionUtils.ts`

**Example output:**
```json
{
  "type": "CURRENCY_NOT_SELECTED",
  "priority": "HIGH",
  "title": "Add EUR to Selected Currencies",
  "description": "EUR has 10 conversions but is not in your selected currencies. Customers may be auto-switched due to location.",
  "actionText": "Go to Settings",
  "icon": "currency"
}
```

---

### 2. `ADD_COUNTRY_TO_MARKET` — Add Country to Shopify Markets

**Trigger conditions (all must be true):**
- `salesRatio >= MIN_SALES_RATIO` (overall average order value)
- Country has sales > 0 but is **not in any enabled Shopify market**
- Country's own `salesRatio` (country sales / country orders) `>= MIN_SALES_RATIO`
- Only the **top** (highest sales) qualifying country is suggested

**Data source:**
- `orders` ClickHouse table → grouped by `country`
- Shopify GraphQL `markets` query → to check which countries are already in markets
- `currencyByCountry` → to find the top currency used in that country

**Implementation:** `findCountriesNotInMarkets()` + threshold checks in `generateSuggestions()`

**Example output:**
```json
{
  "type": "ADD_COUNTRY_TO_MARKET",
  "priority": "HIGH",
  "title": "Add Germany (EUR) to Markets",
  "description": "Germany generated 1200.00 in sales with 15 orders using EUR, but is not in any market. Add it to optimize pricing.",
  "actionText": "Go to Markets",
  "icon": "globe"
}
```

---

### 3. `PRICING_OPTIMIZATION` — Price Increase / Market Split

**Trigger conditions (all must be true):**
- `salesRatio >= MIN_SALES_RATIO`
- Top country (by sales) has `sales > MIN_SALES_RATIO`
- More than 1 unique country exists across all markets (otherwise suggestion is hidden)

**Three sub-cases:**

| Scenario | Market Status | Suggestion |
|----------|--------------|------------|
| Country is **not** in any market | `NOT_IN_MARKET` | "Create Separate Market for {Country}" |
| Country is in a **single-country** market | `SINGLE_COUNTRY_MARKET` | "Increase Prices in {Country} Market" |
| Country is in a **multi-country** market | `MULTI_COUNTRY_MARKET` | "Create Separate Market for {Country}" (split from shared market) |

**Implementation:** `findTopCountryOpportunity()` in `suggestionUtils.ts`

**Post-filter:** `filterAISuggestions()` drops `PRICING_OPTIMIZATION` if total countries across all markets = 1

---

## AI Layer (Gemini)

### Prompts

> **Source file:** [`src/utils/analytics/aiPromptUtils.ts`](https://github.com/user/bucks-analytics-backend/blob/main/src/utils/analytics/aiPromptUtils.ts)

#### System Prompt (`buildSystemPrompt()`)

Instructs Gemini as an "e-commerce analytics consultant" with strict rules:

- Generate **up to 3** suggestions (1 per type)
- Validate each against thresholds before generating
- Return **only** a raw JSON array — no markdown, no explanation
- Suggestion types and their validation order:
  1. **`ADD_CURRENCY`** — `totalConversions >= minConversions` → find top currency not in `selectedCurrencies` with `conversions >= minCurrencySwitches`
  2. **`ADD_COUNTRY_TO_MARKET`** — `salesRatio >= minSalesRatio` → pick best country from `countriesNotInMarkets` whose own `sales/orders >= minSalesRatio`
  3. **`PRICING_OPTIMIZATION`** — `salesRatio >= minSalesRatio` + skip if only 1 country across all markets → check market type (single vs multi-country)

#### User Prompt (`buildUserMessage(analyticsData)`)

Feeds the actual store data into the prompt:

- **Thresholds** — `minConversions`, `minCurrencySwitches`, `minSalesRatio`
- **Overall metrics** — `totalSales`, `totalOrders`, `totalConversions`, `salesRatio`
- **Step 1 data** — `selectedCurrencies`, `currencyConversions` (all currencies + counts)
- **Step 2 data** — `countriesNotInMarkets` (pre-filtered list with `topCurrency`, `sales`, `orders`)
- **Step 3 data** — `locationSales` (sorted by sales), `existingMarkets` (with region country codes)

Each step includes the exact expected JSON format so Gemini returns structured, parseable output.

### Flow

1. **Controller** assembles `aiInputData` — selectedCurrencies, conversions, markets, countriesNotInMarkets, thresholds, etc.
2. **`generateAISuggestions(shop, rangeKey, aiInputData)`** checks **cache** first:
   - Cache table: `ai_suggestions` (MongoDB via Prisma)
   - Cache key: `shop` + `analytics_range` (e.g. `"2025-04-11--2025-05-11"`)
   - Cache TTL: **24 hours** (`expires_at`)
   - If cache is valid → return cached suggestions, `fromCache: true`
3. **Cache miss** → calls `callGeminiAI(analyticsData)`:
   - Model: `gemini-2.0-flash`
   - Temperature: `0.3` (low for deterministic output)
   - Prompt built from `buildSystemPrompt()` + `buildUserMessage(analyticsData)`
   - Response is parsed as JSON array
4. **AI response is post-filtered** through `filterAISuggestions()`:
   - Only **1** `ADD_COUNTRY_TO_MARKET` suggestion allowed
   - `PRICING_OPTIMIZATION` dropped if only 1 country across all markets
5. Country codes in AI text are expanded to full names (e.g. `DE` → `Germany`)
6. **Result stored** in `ai_suggestions` table for caching

### Fallback

If the AI call fails (network error, parse error, missing API key), the controller catches the error and falls back to `manualSuggestions` silently:

```ts
const allSuggestions = aiSuggestions.length > 0 ? aiSuggestions : manualSuggestions;
```

---

## Frontend Integration

The frontend (`pages/analytics/index.jsx`) calls the suggestions API through the same proxy pattern used by other analytics endpoints:

```
GET /api/v1/analytics/proxy?path=suggestions&startDate=...&endDate=...
```

This runs **in parallel** with the existing `summary`, `trends`, `revenue-by-currency`, and `sales-by-country` calls via `Promise.all`.

**Proxy requirement:** `"suggestions"` must be added to the `ALLOWED_PATHS` array in `pages/api/v1/analytics/proxy.js`.

---

## Data Flow Summary

```
1. Frontend loads analytics page
2. Promise.all fires 5 parallel calls (summary, trends, revenue, country, suggestions)
3. Proxy authenticates session → forwards to analytics backend
4. suggestions endpoint:
   a. Fetches shop settings + access token from Prisma
   b. Fetches Shopify markets via GraphQL
   c. Fetches 7 ClickHouse aggregations in parallel
   d. Runs manual rule-based suggestion generation
   e. Checks AI cache → calls Gemini if cache miss
   f. Post-filters AI output
   g. Returns { suggestions (AI preferred), manualSuggestions (fallback) }
5. Frontend renders suggestion cards
```
