# Monthly Analytics Report Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a monthly automated analytics email report sent to Pro plan merchants on the 1st of every month via Brevo, triggered by Google Cloud Scheduler hitting `POST /api/analytics/cron/monthly-report`.

**Architecture:** Extract existing analytics controller logic into a shared pure service layer; a new `monthlyReportService` orchestrates fetch → render → send for each eligible Pro merchant in batches of 10; a `cronAuth` middleware secures the endpoint with `CRON_SECRET`; email HTML is composed from partial builders and delivered via a new Brevo `mailService`. A `dryRun` + single-`shop` mode serves as the testing API (no Cloud Scheduler needed during development).

**Tech Stack:** TypeScript, Express 5, Prisma (MongoDB), ClickHouse, Brevo REST API (`https://api.brevo.com/v3/smtp/email`), Vitest

---

## File Map

### New files
| Path | Responsibility |
|------|---------------|
| `src/services/analytics/analyticsService.ts` | Pure async functions: `getSummaryData`, `getTrendsData`, `getRevenueByCurrencyData`, `getSalesByCountryData` — extracted from controllers |
| `src/services/report/monthlyReportData.ts` | `fetchMonthlyReportPayload(shop, startDate, endDate)` — calls analyticsService + suggestionsService in parallel |
| `src/services/report/dateRange.ts` | `getLastMonthRange()` → `{ startDate, endDate, periodLabel }` |
| `src/services/report/eligibility.ts` | `getEligibleProUsers()`, `getUserByShop(shop)`, `markReportSent(shop)` — MongoDB queries |
| `src/services/report/monthlyReportService.ts` | `runMonthlyReportJob(opts)` — batch loop: fetch → render → send → mark sent |
| `src/email/partials/statsCardsHtml.ts` | `renderStatsCards(summary)` → HTML string |
| `src/email/partials/conversionTrendsHtml.ts` | `renderConversionTrends(trends)` → HTML string |
| `src/email/partials/revenueByCurrencyHtml.ts` | `renderRevenueByCurrency(revenue, defaultCurrency)` → HTML string |
| `src/email/partials/performanceTablesHtml.ts` | `renderPerformanceTables(countries)` → HTML string |
| `src/email/partials/recommendationsHtml.ts` | `renderRecommendations(suggestions)` → HTML string |
| `src/email/renderMonthlyReportHtml.ts` | `renderMonthlyReportHtml(payload, user)` — assembles all partials |
| `src/services/mail/mailService.ts` | `sendMonthlyReport(user, html, periodLabel)` — Brevo REST call |
| `src/middleware/cronAuth.ts` | `cronAuth` middleware — validates `Authorization: Bearer CRON_SECRET` |
| `src/controllers/monthlyReportController.ts` | `runMonthlyReport(req, res)` — HTTP handler |

### Modified files
| Path | Change |
|------|--------|
| `src/controllers/analyticsFetchController.ts` | Delegate to `analyticsService.*`; keep same HTTP interface |
| `src/routes/analyticsRoutes.ts` | Add `router.post("/cron/monthly-report", cronAuth, runMonthlyReport)` |
| `prisma/schema.prisma` | Add `lastMonthlyReportSentAt`, `oneTimeAnalyticsReportSentAt`, `monthlyReportOptOut` to `users` |
| `.env.example` | Add `CRON_SECRET`, `BREVO_API_KEY`, `SHOPIFY_APP_URL`, `MONTHLY_REPORT_FROM_EMAIL`, `MONTHLY_REPORT_FROM_NAME` |

---

## Task 1: Prisma Schema — Add Report Fields to `users`

> ⚠️ **BLOCKER:** Task 5 (`eligibility.ts`) references `lastMonthlyReportSentAt` and `monthlyReportOptOut` via Prisma. These fields **must** exist in the generated client before Task 5 compiles. Always run Task 1 fully (schema edit + `prisma generate`) before starting Task 5.

**Files:**
- Modify: `prisma/schema.prisma`

- [ ] **Step 1.1: Add fields to `users` model**

In `prisma/schema.prisma`, inside the `users` model (after the last existing field `webPixel_last_attempt`), add:

```prisma
lastMonthlyReportSentAt      DateTime?
oneTimeAnalyticsReportSentAt DateTime?
monthlyReportOptOut          Boolean?  @default(false)
```

- [ ] **Step 1.2: Regenerate Prisma client**

```bash
npx prisma generate
```

Expected output: `✔ Generated Prisma Client` with no errors.

- [ ] **Step 1.3: Commit**

```bash
git add prisma/schema.prisma
git commit -m "chore: add monthly report fields to users model"
```

---

## Task 2: Date Range Utility

**Files:**
- Create: `src/services/report/dateRange.ts`
- Create: `src/services/report/dateRange.test.ts`

- [ ] **Step 2.1: Write failing tests**

Create `src/services/report/dateRange.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from "vitest";
import { getLastMonthRange } from "./dateRange";

describe("getLastMonthRange", () => {
  afterEach(() => vi.useRealTimers());

  it("returns previous calendar month dates for February 2026", () => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2026-03-15T10:00:00Z"));
    const { startDate, endDate, periodLabel } = getLastMonthRange();
    expect(startDate).toBe("2026-02-01");
    expect(endDate).toBe("2026-02-28");
    expect(periodLabel).toBe("February 2026");
  });

  it("returns December when current month is January", () => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2026-01-01T00:00:00Z"));
    const { startDate, endDate, periodLabel } = getLastMonthRange();
    expect(startDate).toBe("2025-12-01");
    expect(endDate).toBe("2025-12-31");
    expect(periodLabel).toBe("December 2025");
  });

  it("handles leap year February correctly", () => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2028-03-01T00:00:00Z"));
    const { startDate, endDate, periodLabel } = getLastMonthRange();
    expect(endDate).toBe("2028-02-29");
  });
});
```

- [ ] **Step 2.2: Run tests — verify FAIL**

```bash
npx vitest run src/services/report/dateRange.test.ts
```

Expected: FAIL — "Cannot find module './dateRange'"

- [ ] **Step 2.3: Implement `dateRange.ts`**

Create `src/services/report/dateRange.ts`:

```typescript
export interface DateRange {
  startDate: string;
  endDate: string;
  periodLabel: string;
}

export function getLastMonthRange(): DateRange {
  const now = new Date();
  const year = now.getUTCMonth() === 0 ? now.getUTCFullYear() - 1 : now.getUTCFullYear();
  const month = now.getUTCMonth() === 0 ? 11 : now.getUTCMonth() - 1; // 0-indexed

  const firstDay = new Date(Date.UTC(year, month, 1));
  const lastDay = new Date(Date.UTC(year, month + 1, 0)); // day 0 of next month = last day of this month

  const pad = (n: number) => String(n).padStart(2, "0");
  const startDate = `${firstDay.getUTCFullYear()}-${pad(firstDay.getUTCMonth() + 1)}-${pad(firstDay.getUTCDate())}`;
  const endDate = `${lastDay.getUTCFullYear()}-${pad(lastDay.getUTCMonth() + 1)}-${pad(lastDay.getUTCDate())}`;

  const periodLabel = firstDay.toLocaleString("en-US", {
    month: "long",
    year: "numeric",
    timeZone: "UTC",
  });

  return { startDate, endDate, periodLabel };
}
```

- [ ] **Step 2.4: Run tests — verify PASS**

```bash
npx vitest run src/services/report/dateRange.test.ts
```

Expected: 3 passing.

- [ ] **Step 2.5: Commit**

```bash
git add src/services/report/dateRange.ts src/services/report/dateRange.test.ts
git commit -m "feat: add getLastMonthRange date utility"
```

---

## Task 3: Extract Analytics Service Layer

> **Architecture note — no circular imports:** `analyticsService.ts` is a fully standalone new file. It reimplements its own `buildFilter` and `getDefaultCurrencyForShop` helpers internally — it does NOT import from any controller. Controllers import FROM the service (one direction only). The existing private helpers (`getQueryParams`, `getDefaultCurrency`) in `analyticsFetchController.ts` are left in place and untouched — they are not exported and not shared. `getSalesByCountryCurrency` is also left completely untouched.

**Files:**
- Create: `src/services/analytics/analyticsService.ts`
- Modify: `src/controllers/analyticsFetchController.ts`

- [ ] **Step 3.1: Create `analyticsService.ts` with pure service functions**

Create `src/services/analytics/analyticsService.ts`:

```typescript
import { clickhouse } from "../../utils/clickhouse";
import { getCurrencyConversionSQL, ensureRatesLoaded } from "../../utils/currencyConverter";
import prisma from "../../utils/prisma";

export interface SummaryData {
  totalVisits: number;
  currencyClicks: number;
  totalSales: number;
  totalOrders: number;
  topCurrency: string;
}

export interface TrendItem {
  currency: string;
  conversions: number;
}

export interface RevenueItem {
  currency: string;
  revenue: number;
}

export interface CountryItem {
  country: string;
  sales: number;
}

// Note: buildFilter always includes shop = {shop: String}. All call sites must
// provide a validated shop string (HTTP routes enforce this via validateShop/validateShopQuery).
function buildFilter(shop: string, startDate?: string, endDate?: string): {
  filterStr: string;
  query_params: Record<string, unknown>;
} {
  const query_params: Record<string, unknown> = { shop };
  let filterStr = " AND shop = {shop: String}";
  if (startDate) {
    filterStr += " AND date >= {startDate: String}";
    query_params.startDate = startDate;
  }
  if (endDate) {
    filterStr += " AND date <= {endDate: String}";
    query_params.endDate = endDate;
  }
  return { filterStr, query_params };
}

export async function getDefaultCurrencyForShop(shop: string): Promise<string> {
  try {
    const shopData = await prisma.users.findUnique({
      where: { myshopify_domain: shop },
      select: { currency: true },
    });
    return shopData?.currency?.toUpperCase() || "USD";
  } catch {
    return "USD";
  }
}

export async function getSummaryData(
  shop: string,
  startDate?: string,
  endDate?: string,
  defaultCurrency?: string
): Promise<SummaryData> {
  const { filterStr, query_params } = buildFilter(shop, startDate, endDate);
  const currency = defaultCurrency ?? await getDefaultCurrencyForShop(shop);
  await ensureRatesLoaded();

  const conversionSQL = getCurrencyConversionSQL("order_amount", "checkout_currency", currency);
  const [visitsResult, clicksResult, salesResult] = await Promise.all([
    clickhouse
      .query({ query: `SELECT sum(visits) as totalVisits FROM visits WHERE 1=1${filterStr}`, query_params, format: "JSONEachRow" })
      .then((r) => r.json<{ totalVisits: number }>()),
    clickhouse
      .query({
        query: `SELECT sum(countSum) as currencyClicks, argMax(currency, countSum) as topCurrency FROM (SELECT currency, sum(count) as countSum FROM currency_switches WHERE 1=1${filterStr} GROUP BY currency)`,
        query_params,
        format: "JSONEachRow",
      })
      .then((r) => r.json<{ currencyClicks: number; topCurrency: string }>()),
    clickhouse
      .query({ query: `SELECT sum(${conversionSQL}) as totalSales, count() as totalOrders FROM orders WHERE 1=1${filterStr}`, query_params, format: "JSONEachRow" })
      .then((r) => r.json<{ totalSales: number; totalOrders: number }>()),
  ]);

  let topCurrency = (clicksResult as any)[0]?.topCurrency || "N/A";
  if (topCurrency === "UNKNOWN") topCurrency = currency;

  return {
    totalVisits: Number((visitsResult as any)[0]?.totalVisits || 0),
    currencyClicks: Number((clicksResult as any)[0]?.currencyClicks || 0),
    totalSales: Number((salesResult as any)[0]?.totalSales || 0),
    totalOrders: Number((salesResult as any)[0]?.totalOrders || 0),
    topCurrency,
  };
}

export async function getTrendsData(
  shop: string,
  startDate?: string,
  endDate?: string,
  defaultCurrency?: string
): Promise<TrendItem[]> {
  const { filterStr, query_params } = buildFilter(shop, startDate, endDate);
  const currency = defaultCurrency ?? await getDefaultCurrencyForShop(shop);

  const query = `
    SELECT normalized_currency as currency, sum(count) as conversions
    FROM (
      SELECT if(currency = 'UNKNOWN', '${currency}', currency) as normalized_currency, count
      FROM currency_switches WHERE 1=1${filterStr}
    )
    GROUP BY normalized_currency ORDER BY conversions DESC
  `;

  return clickhouse.query({ query, query_params, format: "JSONEachRow" }).then((r) => r.json<TrendItem>());
}

export async function getRevenueByCurrencyData(
  shop: string,
  startDate?: string,
  endDate?: string,
  defaultCurrency?: string
): Promise<RevenueItem[]> {
  const { filterStr, query_params } = buildFilter(shop, startDate, endDate);
  const currency = defaultCurrency ?? await getDefaultCurrencyForShop(shop);
  await ensureRatesLoaded();

  const conversionSQL = getCurrencyConversionSQL("order_amount", "checkout_currency", currency);
  const query = `
    SELECT normalized_currency as currency, sum(${conversionSQL}) as revenue
    FROM (
      SELECT if(currency = 'UNKNOWN', '${currency}', currency) as normalized_currency, order_amount, checkout_currency
      FROM orders WHERE 1=1${filterStr}
    )
    GROUP BY normalized_currency ORDER BY revenue DESC
  `;

  return clickhouse.query({ query, query_params, format: "JSONEachRow" }).then((r) => r.json<RevenueItem>());
}

export async function getSalesByCountryData(
  shop: string,
  startDate?: string,
  endDate?: string,
  defaultCurrency?: string
): Promise<CountryItem[]> {
  const { filterStr, query_params } = buildFilter(shop, startDate, endDate);
  const currency = defaultCurrency ?? await getDefaultCurrencyForShop(shop);
  await ensureRatesLoaded();

  const query = `
    SELECT country, sum(${getCurrencyConversionSQL("order_amount", "checkout_currency", currency)}) as sales
    FROM orders WHERE 1=1${filterStr}
    GROUP BY country ORDER BY sales DESC
  `;

  return clickhouse.query({ query, query_params, format: "JSONEachRow" }).then((r) => r.json<CountryItem>());
}
```

- [ ] **Step 3.2: Refactor `analyticsFetchController.ts` to delegate to service**

Replace the body of each export in `src/controllers/analyticsFetchController.ts` so the controllers are thin wrappers. Keep the same function signatures and HTTP responses.

Replace `getSummary`:
```typescript
export const getSummary = async (req: Request, res: Response) => {
  try {
    const shop = (req.query.shop || req.body.shop) as string;
    const startDate = (req.query.startDate || req.body.startDate) as string | undefined;
    const endDate = (req.query.endDate || req.body.endDate) as string | undefined;
    const data = await getSummaryData(shop, startDate, endDate);
    res.status(200).json(data);
  } catch (error) {
    console.error("[BUCKS Analytics] Get Summary Error:", error);
    res.status(500).json({ error: "Failed to fetch summary data" });
  }
};
```

Replace `getTrends`:
```typescript
export const getTrends = async (req: Request, res: Response) => {
  try {
    const shop = (req.query.shop || req.body.shop) as string;
    const startDate = (req.query.startDate || req.body.startDate) as string | undefined;
    const endDate = (req.query.endDate || req.body.endDate) as string | undefined;
    res.status(200).json(await getTrendsData(shop, startDate, endDate));
  } catch (error) {
    console.error("[BUCKS Analytics] Get Trends Error:", error);
    res.status(500).json({ error: "Failed to fetch trends data" });
  }
};
```

Replace `getRevenueByCurrency`:
```typescript
export const getRevenueByCurrency = async (req: Request, res: Response) => {
  try {
    const shop = (req.query.shop || req.body.shop) as string;
    const startDate = (req.query.startDate || req.body.startDate) as string | undefined;
    const endDate = (req.query.endDate || req.body.endDate) as string | undefined;
    res.status(200).json(await getRevenueByCurrencyData(shop, startDate, endDate));
  } catch (error) {
    console.error("[BUCKS Analytics] Get Revenue Error:", error);
    res.status(500).json({ error: "Failed to fetch revenue data" });
  }
};
```

Replace `getSalesByCountry`:
```typescript
export const getSalesByCountry = async (req: Request, res: Response) => {
  try {
    const shop = (req.query.shop || req.body.shop) as string;
    const startDate = (req.query.startDate || req.body.startDate) as string | undefined;
    const endDate = (req.query.endDate || req.body.endDate) as string | undefined;
    res.status(200).json(await getSalesByCountryData(shop, startDate, endDate));
  } catch (error) {
    console.error("[BUCKS Analytics] Get Sales By Country Error:", error);
    res.status(500).json({ error: "Failed to fetch sales by country data" });
  }
};
```

Add imports at the top of `analyticsFetchController.ts`:
```typescript
import {
  getSummaryData,
  getTrendsData,
  getRevenueByCurrencyData,
  getSalesByCountryData,
} from "../services/analytics/analyticsService";
```

- [ ] **Step 3.3: Build to verify no TypeScript errors**

```bash
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 3.4: Commit**

```bash
git add src/services/analytics/analyticsService.ts src/controllers/analyticsFetchController.ts
git commit -m "refactor: extract analytics service layer from controllers"
```

---

## Task 4: Monthly Report Data Fetcher

> **Suggestions fallback safety:** `getSuggestionsForShop` internally calls Gemini AI which can crash if `GOOGLE_SERVICE_ACCOUNT_BASE64` is unset. This is already handled — `suggestionsService.ts` wraps the AI call in a `try/catch` and falls back to manual suggestions. The monthly report will always receive a valid `SuggestionsResult` (possibly empty `suggestions: []`). No extra error handling needed here.

**Files:**
- Create: `src/services/report/monthlyReportData.ts`

- [ ] **Step 4.1: Create `monthlyReportData.ts`**

Create `src/services/report/monthlyReportData.ts`:

```typescript
import { getSummaryData, getTrendsData, getRevenueByCurrencyData, getSalesByCountryData, getDefaultCurrencyForShop, SummaryData, TrendItem, RevenueItem, CountryItem } from "../analytics/analyticsService";
import { getSuggestionsForShop } from "../suggestions/suggestionsService";
import { SuggestionsResult } from "../suggestions/types";

export interface MonthlyReportPayload {
  shop: string;
  periodLabel: string;
  startDate: string;
  endDate: string;
  defaultCurrency: string;
  summary: SummaryData;
  trends: TrendItem[];
  revenue: RevenueItem[];
  countries: CountryItem[];
  suggestions: SuggestionsResult;
}

export async function fetchMonthlyReportPayload(
  shop: string,
  startDate: string,
  endDate: string,
  periodLabel: string
): Promise<MonthlyReportPayload> {
  // Resolve defaultCurrency once — passed to all 4 service functions to avoid
  // 5 redundant Prisma findUnique calls (1 per function) for the same shop.
  const defaultCurrency = await getDefaultCurrencyForShop(shop);

  const [summary, trends, revenue, countries, suggestions] = await Promise.all([
    getSummaryData(shop, startDate, endDate, defaultCurrency),
    getTrendsData(shop, startDate, endDate, defaultCurrency),
    getRevenueByCurrencyData(shop, startDate, endDate, defaultCurrency),
    getSalesByCountryData(shop, startDate, endDate, defaultCurrency),
    getSuggestionsForShop(shop, startDate, endDate),
  ]);

  return { shop, periodLabel, startDate, endDate, defaultCurrency, summary, trends, revenue, countries, suggestions };
}
```

- [ ] **Step 4.2: Build check**

```bash
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 4.3: Commit**

```bash
git add src/services/report/monthlyReportData.ts
git commit -m "feat: add fetchMonthlyReportPayload for report data aggregation"
```

---

## Task 5: Eligibility Queries (MongoDB)

**Files:**
- Create: `src/services/report/eligibility.ts`
- Create: `src/services/report/eligibility.test.ts`

- [ ] **Step 5.1: Write failing tests**

Create `src/services/report/eligibility.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";

vi.mock("../../utils/prisma", () => ({
  default: {
    users: {
      findMany: vi.fn(),
      findUnique: vi.fn(),
      update: vi.fn(),
    },
  },
}));

import prisma from "../../utils/prisma";
import { getEligibleProUsers, getUserByShop, markReportSent } from "./eligibility";

const PRO_PLANS = [
  "bucks_premium_pro",
  "bucks_premium_pro_annual_60",
  "bucks_premium_pro_annual_65",
];

describe("getEligibleProUsers", () => {
  beforeEach(() => vi.clearAllMocks());

  it("calls findMany with correct plan filter and active filter", async () => {
    (prisma.users.findMany as any).mockResolvedValue([]);
    await getEligibleProUsers();
    const call = (prisma.users.findMany as any).mock.calls[0][0];
    expect(call.where.bucks_plan.in).toEqual(PRO_PLANS);
    expect(call.where.is_active).toBe(true);
  });

  it("returns array of users", async () => {
    const mockUsers = [{ myshopify_domain: "a.myshopify.com", email: "a@b.com" }];
    (prisma.users.findMany as any).mockResolvedValue(mockUsers);
    const result = await getEligibleProUsers();
    expect(result).toEqual(mockUsers);
  });
});

describe("getUserByShop", () => {
  beforeEach(() => vi.clearAllMocks());

  it("returns array with single user when found", async () => {
    const user = { myshopify_domain: "test.myshopify.com", email: "t@e.com" };
    (prisma.users.findUnique as any).mockResolvedValue(user);
    const result = await getUserByShop("test.myshopify.com");
    expect(result).toEqual([user]);
  });

  it("returns empty array when user not found", async () => {
    (prisma.users.findUnique as any).mockResolvedValue(null);
    const result = await getUserByShop("notfound.myshopify.com");
    expect(result).toEqual([]);
  });
});

describe("markReportSent", () => {
  beforeEach(() => vi.clearAllMocks());

  it("updates lastMonthlyReportSentAt for the shop", async () => {
    (prisma.users.update as any).mockResolvedValue({});
    await markReportSent("shop.myshopify.com");
    expect(prisma.users.update).toHaveBeenCalledWith(
      expect.objectContaining({
        where: { myshopify_domain: "shop.myshopify.com" },
        data: expect.objectContaining({ lastMonthlyReportSentAt: expect.any(Date) }),
      })
    );
  });
});
```

- [ ] **Step 5.2: Run tests — verify FAIL**

```bash
npx vitest run src/services/report/eligibility.test.ts
```

Expected: FAIL — "Cannot find module './eligibility'"

- [ ] **Step 5.3: Implement `eligibility.ts`**

Create `src/services/report/eligibility.ts`:

```typescript
import prisma from "../../utils/prisma";

export const PRO_ANALYTICS_PLANS = [
  "bucks_premium_pro",
  "bucks_premium_pro_annual_60",
  "bucks_premium_pro_annual_65",
];

export interface EligibleUser {
  myshopify_domain: string;
  email: string | null;
  customer_email: string | null;
  shop_owner: string | null;
  name: string | null;
  lastMonthlyReportSentAt: Date | null;
  monthlyReportOptOut: boolean | null;
}

export async function getEligibleProUsers(): Promise<EligibleUser[]> {
  const startOfCurrentMonth = new Date();
  startOfCurrentMonth.setUTCDate(1);
  startOfCurrentMonth.setUTCHours(0, 0, 0, 0);

  return prisma.users.findMany({
    where: {
      bucks_plan: { in: PRO_ANALYTICS_PLANS },
      is_active: true,
      monthlyReportOptOut: { not: true },
      AND: [
        // At least one email field must be present — excludes no-email users at DB level
        { OR: [{ email: { not: "" } }, { customer_email: { not: "" } }] },
        // Idempotency: not yet sent this calendar month
        { OR: [{ lastMonthlyReportSentAt: null }, { lastMonthlyReportSentAt: { lt: startOfCurrentMonth } }] },
      ],
    },
    select: {
      myshopify_domain: true,
      email: true,
      customer_email: true,
      shop_owner: true,
      name: true,
      lastMonthlyReportSentAt: true,
      monthlyReportOptOut: true,
    },
  }) as Promise<EligibleUser[]>;
}

export async function getUserByShop(shop: string): Promise<EligibleUser[]> {
  const user = await prisma.users.findUnique({
    where: { myshopify_domain: shop },
    select: {
      myshopify_domain: true,
      email: true,
      customer_email: true,
      shop_owner: true,
      name: true,
      lastMonthlyReportSentAt: true,
      monthlyReportOptOut: true,
    },
  });
  return user ? [user as EligibleUser] : [];
}

export async function markReportSent(shop: string): Promise<void> {
  await prisma.users.update({
    where: { myshopify_domain: shop },
    data: { lastMonthlyReportSentAt: new Date() },
  });
}
```

- [ ] **Step 5.4: Run tests — verify PASS**

```bash
npx vitest run src/services/report/eligibility.test.ts
```

Expected: 5 passing.

- [ ] **Step 5.5: Commit**

```bash
git add src/services/report/eligibility.ts src/services/report/eligibility.test.ts
git commit -m "feat: add eligibility queries for monthly report"
```

---

## Task 6: Email HTML Partials

**Files:**
- Create: `src/email/partials/statsCardsHtml.ts`
- Create: `src/email/partials/conversionTrendsHtml.ts`
- Create: `src/email/partials/revenueByCurrencyHtml.ts`
- Create: `src/email/partials/performanceTablesHtml.ts`
- Create: `src/email/partials/recommendationsHtml.ts`

> **Note:** Email HTML must use inline styles and table-based layouts only. No external CSS, no classes that depend on stylesheets, no `<style>` blocks relying on class selectors (Outlook strips them). Use `bgcolor`, `cellpadding`, `cellspacing` on `<table>` elements.

- [ ] **Step 6.1: Create `statsCardsHtml.ts`**

Create `src/email/partials/statsCardsHtml.ts`:

```typescript
import { SummaryData } from "../../services/analytics/analyticsService";

function formatNumber(n: number): string {
  if (n >= 1_000_000) return (n / 1_000_000).toFixed(1) + "M";
  if (n >= 1_000) return (n / 1_000).toFixed(1) + "K";
  return String(Math.round(n));
}

function statCard(title: string, value: string): string {
  return `
    <td width="50%" style="padding:8px;">
      <table width="100%" cellpadding="0" cellspacing="0" border="0"
        style="background:#f9f9f9;border:1px solid #e8e8e8;border-radius:12px;">
        <tr>
          <td style="padding:16px 20px;">
            <div style="font-size:11px;color:#303030A6;font-family:Arial,sans-serif;margin-bottom:6px;text-transform:uppercase;letter-spacing:0.5px;">${title}</div>
            <div style="font-size:22px;font-weight:700;color:#303030;font-family:Arial,sans-serif;">${value}</div>
          </td>
        </tr>
      </table>
    </td>`;
}

export function renderStatsCards(summary: SummaryData): string {
  return `
    <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:24px;">
      <tr>
        ${statCard("Total Visits", formatNumber(Number(summary.totalVisits)))}
        ${statCard("Currency Clicks", formatNumber(Number(summary.currencyClicks)))}
      </tr>
      <tr>
        ${statCard("Total Sales", formatNumber(Number(summary.totalSales)))}
        ${statCard("Top Currency", summary.topCurrency || "N/A")}
      </tr>
    </table>`;
}
```

- [ ] **Step 6.2: Create `conversionTrendsHtml.ts`**

Create `src/email/partials/conversionTrendsHtml.ts`:

```typescript
import { TrendItem } from "../../services/analytics/analyticsService";

const CURRENCY_COLORS: Record<string, string> = {
  USD: "#008060", EUR: "#1a73e8", GBP: "#7b2d8b", CAD: "#e67e22",
  AUD: "#e74c3c", JPY: "#2ecc71", CHF: "#3498db", CNY: "#c0392b",
};

function getColor(currency: string): string {
  return CURRENCY_COLORS[currency.toUpperCase()] || "#8c8c8c";
}

export function renderConversionTrends(trends: TrendItem[]): string {
  if (!trends || trends.length === 0) return "";

  const top5 = trends.slice(0, 5);
  const maxVal = Math.max(...top5.map((t) => Number(t.conversions)));

  const rows = top5
    .map((t) => {
      const pct = maxVal > 0 ? Math.round((Number(t.conversions) / maxVal) * 100) : 0;
      const color = getColor(t.currency);
      return `
      <tr>
        <td style="padding:6px 0;font-family:Arial,sans-serif;font-size:13px;color:#303030;width:50px;">${t.currency}</td>
        <td style="padding:6px 8px;">
          <table width="100%" cellpadding="0" cellspacing="0">
            <tr>
              <td width="${pct}%" bgcolor="${color}" style="height:16px;border-radius:4px;font-size:1px;">&nbsp;</td>
              <td width="${100 - pct}%">&nbsp;</td>
            </tr>
          </table>
        </td>
        <td style="padding:6px 0;font-family:Arial,sans-serif;font-size:13px;color:#303030;text-align:right;width:60px;">${Number(t.conversions).toLocaleString()}</td>
      </tr>`;
    })
    .join("");

  return `
    <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:24px;">
      <tr><td style="font-size:16px;font-weight:700;color:#303030;font-family:Arial,sans-serif;padding-bottom:12px;">Currency Conversion Trends</td></tr>
      <tr><td><table width="100%" cellpadding="0" cellspacing="0">${rows}</table></td></tr>
    </table>`;
}
```

- [ ] **Step 6.3: Create `revenueByCurrencyHtml.ts`**

Create `src/email/partials/revenueByCurrencyHtml.ts`:

```typescript
import { RevenueItem } from "../../services/analytics/analyticsService";

export function renderRevenueByCurrency(revenue: RevenueItem[], defaultCurrency: string): string {
  if (!revenue || revenue.length === 0) return "";

  const total = revenue.reduce((s, r) => s + Number(r.revenue), 0);
  const top5 = revenue.slice(0, 5);

  const rows = top5
    .map((r) => {
      const pct = total > 0 ? Math.round((Number(r.revenue) / total) * 100) : 0;
      return `
      <tr style="border-bottom:1px solid #f0f0f0;">
        <td style="padding:10px 0;font-family:Arial,sans-serif;font-size:13px;color:#303030;">${r.currency}</td>
        <td style="padding:10px 8px;">
          <table width="100%" cellpadding="0" cellspacing="0">
            <tr>
              <td width="${pct}%" bgcolor="#008060" style="height:12px;border-radius:3px;font-size:1px;">&nbsp;</td>
              <td width="${100 - pct}%">&nbsp;</td>
            </tr>
          </table>
        </td>
        <td style="padding:10px 0;font-family:Arial,sans-serif;font-size:13px;color:#303030;text-align:right;">${pct}%</td>
        <td style="padding:10px 0 10px 12px;font-family:Arial,sans-serif;font-size:13px;color:#303030;text-align:right;">${Number(r.revenue).toFixed(2)} ${defaultCurrency}</td>
      </tr>`;
    })
    .join("");

  return `
    <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:24px;">
      <tr><td style="font-size:16px;font-weight:700;color:#303030;font-family:Arial,sans-serif;padding-bottom:12px;">Revenue by Currency</td></tr>
      <tr><td><table width="100%" cellpadding="0" cellspacing="0">${rows}</table></td></tr>
    </table>`;
}
```

- [ ] **Step 6.4: Create `performanceTablesHtml.ts`**

Create `src/email/partials/performanceTablesHtml.ts`:

```typescript
import { CountryItem } from "../../services/analytics/analyticsService";

export function renderPerformanceTables(countries: CountryItem[], defaultCurrency: string): string {
  if (!countries || countries.length === 0) return "";

  const top5 = countries.slice(0, 5);

  const rows = top5
    .map(
      (c) => `
    <tr style="border-bottom:1px solid #f0f0f0;">
      <td style="padding:10px 0;font-family:Arial,sans-serif;font-size:13px;color:#303030;">${c.country || "Unknown"}</td>
      <td style="padding:10px 0;font-family:Arial,sans-serif;font-size:13px;color:#303030;text-align:right;">${Number(c.sales).toFixed(2)} ${defaultCurrency}</td>
    </tr>`
    )
    .join("");

  return `
    <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:24px;">
      <tr><td style="font-size:16px;font-weight:700;color:#303030;font-family:Arial,sans-serif;padding-bottom:12px;">Top Sales by Country</td></tr>
      <tr>
        <td>
          <table width="100%" cellpadding="0" cellspacing="0">
            <tr style="border-bottom:2px solid #e8e8e8;">
              <td style="padding:8px 0;font-family:Arial,sans-serif;font-size:11px;color:#303030A6;text-transform:uppercase;">Country</td>
              <td style="padding:8px 0;font-family:Arial,sans-serif;font-size:11px;color:#303030A6;text-transform:uppercase;text-align:right;">Sales (${defaultCurrency})</td>
            </tr>
            ${rows}
          </table>
        </td>
      </tr>
    </table>`;
}
```

- [ ] **Step 6.5: Create `recommendationsHtml.ts`**

Create `src/email/partials/recommendationsHtml.ts`:

```typescript
import { SuggestionsResult } from "../../services/suggestions/types";

export function renderRecommendations(suggestions: SuggestionsResult): string {
  const items = suggestions?.suggestions;
  if (!items || items.length === 0) return "";

  const top3 = items.slice(0, 3);

  const cards = top3
    .map(
      (s) => `
    <tr>
      <td style="padding:0 0 12px 0;">
        <table width="100%" cellpadding="0" cellspacing="0"
          style="background:#f0faf6;border-left:4px solid #008060;border-radius:0 8px 8px 0;">
          <tr>
            <td style="padding:14px 16px;">
              <div style="font-size:13px;font-weight:700;color:#303030;font-family:Arial,sans-serif;margin-bottom:4px;">${s.title || "Recommendation"}</div>
              <div style="font-size:12px;color:#303030A6;font-family:Arial,sans-serif;line-height:1.5;">${s.description || ""}</div>
            </td>
          </tr>
        </table>
      </td>
    </tr>`
    )
    .join("");

  return `
    <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:24px;">
      <tr><td style="font-size:16px;font-weight:700;color:#303030;font-family:Arial,sans-serif;padding-bottom:12px;">Smart Recommendations</td></tr>
      <tr><td><table width="100%" cellpadding="0" cellspacing="0">${cards}</table></td></tr>
    </table>`;
}
```

- [ ] **Step 6.6: Build check**

```bash
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 6.7: Commit**

```bash
git add src/email/partials/
git commit -m "feat: add email HTML partial renderers"
```

---

## Task 7: Main Email Template Assembler

**Files:**
- Create: `src/email/renderMonthlyReportHtml.ts`
- Create: `src/email/renderMonthlyReportHtml.test.ts`

- [ ] **Step 7.1: Write failing tests**

Create `src/email/renderMonthlyReportHtml.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { renderMonthlyReportHtml } from "./renderMonthlyReportHtml";
import { MonthlyReportPayload } from "../services/report/monthlyReportData";
import { EligibleUser } from "../services/report/eligibility";

const mockPayload: MonthlyReportPayload = {
  shop: "test.myshopify.com",
  periodLabel: "May 2026",
  startDate: "2026-05-01",
  endDate: "2026-05-31",
  defaultCurrency: "USD",
  summary: { totalVisits: 1000, currencyClicks: 200, totalSales: 5000, totalOrders: 50, topCurrency: "EUR" },
  trends: [{ currency: "EUR", conversions: 150 }],
  revenue: [{ currency: "EUR", revenue: 3000 }],
  countries: [{ country: "DE", sales: 2000 }],
  suggestions: { suggestions: [], aiFromCache: false, usingAI: false },
};

const mockUser: EligibleUser = {
  myshopify_domain: "test.myshopify.com",
  email: "owner@test.com",
  customer_email: "",
  shop_owner: "John Doe",
  name: "Test Shop",
  lastMonthlyReportSentAt: null,
  monthlyReportOptOut: false,
};

describe("renderMonthlyReportHtml", () => {
  it("returns a non-empty HTML string", () => {
    const html = renderMonthlyReportHtml(mockPayload, mockUser);
    expect(typeof html).toBe("string");
    expect(html.length).toBeGreaterThan(200);
  });

  it("includes the period label", () => {
    const html = renderMonthlyReportHtml(mockPayload, mockUser);
    expect(html).toContain("May 2026");
  });

  it("includes the shop owner name", () => {
    const html = renderMonthlyReportHtml(mockPayload, mockUser);
    expect(html).toContain("John");
  });

  it("includes a CTA link", () => {
    const html = renderMonthlyReportHtml(mockPayload, mockUser);
    expect(html).toContain("View full analytics");
  });

  it("renders stat cards with visit count", () => {
    const html = renderMonthlyReportHtml(mockPayload, mockUser);
    expect(html).toContain("1.0K");
  });
});
```

- [ ] **Step 7.2: Run tests — verify FAIL**

```bash
npx vitest run src/email/renderMonthlyReportHtml.test.ts
```

Expected: FAIL — "Cannot find module './renderMonthlyReportHtml'"

- [ ] **Step 7.3: Implement `renderMonthlyReportHtml.ts`**

Create `src/email/renderMonthlyReportHtml.ts`:

```typescript
import { MonthlyReportPayload } from "../services/report/monthlyReportData";
import { EligibleUser } from "../services/report/eligibility";
import { renderStatsCards } from "./partials/statsCardsHtml";
import { renderConversionTrends } from "./partials/conversionTrendsHtml";
import { renderRevenueByCurrency } from "./partials/revenueByCurrencyHtml";
import { renderPerformanceTables } from "./partials/performanceTablesHtml";
import { renderRecommendations } from "./partials/recommendationsHtml";

const SHOPIFY_APP_URL = process.env.SHOPIFY_APP_URL || "https://apps.shopify.com/bucks";

export function renderMonthlyReportHtml(payload: MonthlyReportPayload, user: EligibleUser): string {
  const firstName = (user.shop_owner || "Merchant").split(" ")[0];
  const shopName = user.name || user.myshopify_domain;
  const ctaUrl = `${SHOPIFY_APP_URL}/analytics?shop=${encodeURIComponent(user.myshopify_domain)}`;

  const statsSection = renderStatsCards(payload.summary);
  const trendsSection = renderConversionTrends(payload.trends);
  const revenueSection = renderRevenueByCurrency(payload.revenue, payload.defaultCurrency);
  const tablesSection = renderPerformanceTables(payload.countries, payload.defaultCurrency);
  const recommendationsSection = renderRecommendations(payload.suggestions);

  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Your BUCKS Analytics Report — ${payload.periodLabel}</title>
</head>
<body style="margin:0;padding:0;background:#f4f4f4;font-family:Arial,sans-serif;">
  <table width="100%" cellpadding="0" cellspacing="0" border="0" bgcolor="#f4f4f4">
    <tr>
      <td align="center" style="padding:32px 16px;">
        <table width="600" cellpadding="0" cellspacing="0" border="0"
          style="max-width:600px;background:#ffffff;border-radius:12px;overflow:hidden;border:1px solid #e8e8e8;">

          <!-- Header -->
          <tr>
            <td bgcolor="#008060" style="padding:24px 32px;text-align:center;">
              <div style="font-size:24px;font-weight:700;color:#ffffff;font-family:Arial,sans-serif;letter-spacing:-0.5px;">BUCKS</div>
              <div style="font-size:14px;color:rgba(255,255,255,0.85);margin-top:4px;font-family:Arial,sans-serif;">Your ${payload.periodLabel} Analytics Report</div>
            </td>
          </tr>

          <!-- Greeting -->
          <tr>
            <td style="padding:28px 32px 16px 32px;">
              <div style="font-size:18px;font-weight:600;color:#303030;font-family:Arial,sans-serif;margin-bottom:8px;">Hi ${firstName},</div>
              <div style="font-size:14px;color:#303030A6;font-family:Arial,sans-serif;line-height:1.6;">Here's how <strong>${shopName}</strong> performed with BUCKS last month.</div>
            </td>
          </tr>

          <!-- Stats Cards -->
          <tr><td style="padding:8px 32px;">${statsSection}</td></tr>

          <!-- Conversion Trends -->
          ${trendsSection ? `<tr><td style="padding:8px 32px;">${trendsSection}</td></tr>` : ""}

          <!-- Revenue by Currency -->
          ${revenueSection ? `<tr><td style="padding:8px 32px;">${revenueSection}</td></tr>` : ""}

          <!-- Performance Tables -->
          ${tablesSection ? `<tr><td style="padding:8px 32px;">${tablesSection}</td></tr>` : ""}

          <!-- Recommendations -->
          ${recommendationsSection ? `<tr><td style="padding:8px 32px;">${recommendationsSection}</td></tr>` : ""}

          <!-- CTA -->
          <tr>
            <td style="padding:24px 32px 32px 32px;text-align:center;">
              <a href="${ctaUrl}"
                style="display:inline-block;background:#008060;color:#ffffff;font-family:Arial,sans-serif;font-size:15px;font-weight:600;text-decoration:none;padding:14px 32px;border-radius:8px;">
                View full analytics →
              </a>
            </td>
          </tr>

          <!-- Footer -->
          <tr>
            <td bgcolor="#f9f9f9" style="padding:20px 32px;border-top:1px solid #e8e8e8;text-align:center;">
              <div style="font-size:11px;color:#303030A6;font-family:Arial,sans-serif;line-height:1.6;">
                You're receiving this because you're on the BUCKS Pro Analytics plan.<br>
                © ${new Date().getFullYear()} BUCKS · <a href="${SHOPIFY_APP_URL}" style="color:#008060;text-decoration:none;">bucks.helixo.co</a>
              </div>
            </td>
          </tr>

        </table>
      </td>
    </tr>
  </table>
</body>
</html>`;
}
```

- [ ] **Step 7.4: Run tests — verify PASS**

```bash
npx vitest run src/email/renderMonthlyReportHtml.test.ts
```

Expected: 5 passing.

- [ ] **Step 7.5: Commit**

```bash
git add src/email/renderMonthlyReportHtml.ts src/email/renderMonthlyReportHtml.test.ts
git commit -m "feat: add monthly report HTML email assembler"
```

---

## Task 8: Brevo Mail Service

**Files:**
- Create: `src/services/mail/mailService.ts`
- Create: `src/services/mail/mailService.test.ts`

- [ ] **Step 8.1: Write failing tests**

Create `src/services/mail/mailService.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";

const mockFetch = vi.fn();
vi.stubGlobal("fetch", mockFetch);

import { sendMonthlyReport } from "./mailService";
import { EligibleUser } from "../report/eligibility";

const mockUser: EligibleUser = {
  myshopify_domain: "test.myshopify.com",
  email: "owner@test.com",
  customer_email: "",
  shop_owner: "John Doe",
  name: "Test Shop",
  lastMonthlyReportSentAt: null,
  monthlyReportOptOut: false,
};

describe("sendMonthlyReport", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    process.env.BREVO_API_KEY = "test-api-key";
    process.env.MONTHLY_REPORT_FROM_EMAIL = "reports@bucks.com";
    process.env.MONTHLY_REPORT_FROM_NAME = "BUCKS";
  });

  afterEach(() => {
    delete process.env.BREVO_API_KEY;
  });

  it("calls Brevo smtp/email endpoint with correct data", async () => {
    mockFetch.mockResolvedValueOnce({ ok: true, json: async () => ({ messageId: "abc" }) });

    await sendMonthlyReport(mockUser, "<html>test</html>", "May 2026");

    expect(mockFetch).toHaveBeenCalledWith(
      "https://api.brevo.com/v3/smtp/email",
      expect.objectContaining({
        method: "POST",
        headers: expect.objectContaining({ "api-key": "test-api-key" }),
      })
    );

    const body = JSON.parse(mockFetch.mock.calls[0][1].body);
    expect(body.to[0].email).toBe("owner@test.com");
    expect(body.subject).toContain("May 2026");
    expect(body.htmlContent).toBe("<html>test</html>");
  });

  it("uses customer_email when email is empty", async () => {
    mockFetch.mockResolvedValueOnce({ ok: true, json: async () => ({}) });
    const user = { ...mockUser, email: "", customer_email: "cust@test.com" };
    await sendMonthlyReport(user, "<html/>", "May 2026");
    const body = JSON.parse(mockFetch.mock.calls[0][1].body);
    expect(body.to[0].email).toBe("cust@test.com");
  });

  it("throws when no email is available", async () => {
    const user = { ...mockUser, email: "", customer_email: "" };
    await expect(sendMonthlyReport(user, "<html/>", "May 2026")).rejects.toThrow("No email");
  });

  it("throws when Brevo returns non-ok response", async () => {
    mockFetch.mockResolvedValueOnce({ ok: false, json: async () => ({ message: "Unauthorized" }) });
    await expect(sendMonthlyReport(mockUser, "<html/>", "May 2026")).rejects.toThrow();
  });
});
```

- [ ] **Step 8.2: Run tests — verify FAIL**

```bash
npx vitest run src/services/mail/mailService.test.ts
```

Expected: FAIL — "Cannot find module './mailService'"

- [ ] **Step 8.3: Implement `mailService.ts`**

Create `src/services/mail/mailService.ts`:

```typescript
import { EligibleUser } from "../report/eligibility";

const BREVO_API_URL = "https://api.brevo.com/v3/smtp/email";

export async function sendMonthlyReport(
  user: EligibleUser,
  html: string,
  periodLabel: string
): Promise<void> {
  const apiKey = process.env.BREVO_API_KEY;
  if (!apiKey) throw new Error("BREVO_API_KEY not configured");

  const recipientEmail = user.email || user.customer_email;
  if (!recipientEmail) throw new Error(`No email address for shop ${user.myshopify_domain}`);

  const fromEmail = process.env.MONTHLY_REPORT_FROM_EMAIL || "reports@bucks.com";
  const fromName = process.env.MONTHLY_REPORT_FROM_NAME || "BUCKS";
  const firstName = (user.shop_owner || "Merchant").split(" ")[0];

  const payload = {
    sender: { name: fromName, email: fromEmail },
    to: [{ email: recipientEmail, name: user.shop_owner || "Merchant" }],
    subject: `Your BUCKS Analytics Report — ${periodLabel}`,
    htmlContent: html,
    params: { FIRSTNAME: firstName, SHOP: user.myshopify_domain },
  };

  const response = await fetch(BREVO_API_URL, {
    method: "POST",
    headers: {
      accept: "application/json",
      "content-type": "application/json",
      "api-key": apiKey,
    },
    body: JSON.stringify(payload),
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new Error(`Brevo error for ${user.myshopify_domain}: ${JSON.stringify(error)}`);
  }

  console.log(`[MailService] Sent monthly report to ${recipientEmail} (${user.myshopify_domain})`);
}
```

- [ ] **Step 8.4: Run tests — verify PASS**

```bash
npx vitest run src/services/mail/mailService.test.ts
```

Expected: 4 passing.

- [ ] **Step 8.5: Commit**

```bash
git add src/services/mail/mailService.ts src/services/mail/mailService.test.ts
git commit -m "feat: add Brevo mail service for monthly report sending"
```

---

## Task 9: Cron Auth Middleware

**Files:**
- Create: `src/middleware/cronAuth.ts`
- Create: `src/middleware/cronAuth.test.ts`

- [ ] **Step 9.1: Write failing tests**

Create `src/middleware/cronAuth.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import { cronAuth } from "./cronAuth";

function makeReqRes(authHeader?: string) {
  const req: any = { headers: { authorization: authHeader } };
  const res: any = {
    status: vi.fn().mockReturnThis(),
    json: vi.fn().mockReturnThis(),
  };
  const next = vi.fn();
  return { req, res, next };
}

describe("cronAuth", () => {
  beforeEach(() => {
    process.env.CRON_SECRET = "supersecret";
  });

  afterEach(() => {
    process.env.CRON_SECRET = "supersecret";
  });

  it("calls next() when Authorization header matches", () => {
    const { req, res, next } = makeReqRes("Bearer supersecret");
    cronAuth(req, res, next);
    expect(next).toHaveBeenCalledOnce();
    expect(res.status).not.toHaveBeenCalled();
  });

  it("returns 401 when no Authorization header", () => {
    const { req, res, next } = makeReqRes(undefined);
    cronAuth(req, res, next);
    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });

  it("returns 401 when secret is wrong", () => {
    const { req, res, next } = makeReqRes("Bearer wrongsecret");
    cronAuth(req, res, next);
    expect(res.status).toHaveBeenCalledWith(401);
    expect(next).not.toHaveBeenCalled();
  });

  it("returns 401 when Bearer prefix is missing", () => {
    const { req, res, next } = makeReqRes("supersecret");
    cronAuth(req, res, next);
    expect(res.status).toHaveBeenCalledWith(401);
  });

  it("returns 500 when CRON_SECRET env var is not set", () => {
    delete process.env.CRON_SECRET;
    const { req, res, next } = makeReqRes("Bearer anything");
    cronAuth(req, res, next);
    expect(res.status).toHaveBeenCalledWith(500);
  });
});
```

- [ ] **Step 9.2: Run tests — verify FAIL**

```bash
npx vitest run src/middleware/cronAuth.test.ts
```

Expected: FAIL — "Cannot find module './cronAuth'"

- [ ] **Step 9.3: Implement `cronAuth.ts`**

Create `src/middleware/cronAuth.ts`:

```typescript
import { Request, Response, NextFunction } from "express";
import { timingSafeEqual } from "crypto";

export function cronAuth(req: Request, res: Response, next: NextFunction): void {
  const secret = process.env.CRON_SECRET;
  if (!secret) {
    res.status(500).json({ error: "CRON_SECRET not configured" });
    return;
  }

  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }

  const provided = authHeader.slice(7);
  const providedBuf = Buffer.from(provided);
  const secretBuf = Buffer.from(secret);

  if (
    providedBuf.length !== secretBuf.length ||
    !timingSafeEqual(providedBuf, secretBuf)
  ) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }

  next();
}
```

- [ ] **Step 9.4: Run tests — verify PASS**

```bash
npx vitest run src/middleware/cronAuth.test.ts
```

Expected: 5 passing.

- [ ] **Step 9.5: Commit**

```bash
git add src/middleware/cronAuth.ts src/middleware/cronAuth.test.ts
git commit -m "feat: add cronAuth middleware with timing-safe secret comparison"
```

---

## Task 10: Monthly Report Service (Orchestrator)

**Files:**
- Create: `src/services/report/monthlyReportService.ts`

- [ ] **Step 10.1: Create `monthlyReportService.ts`**

Create `src/services/report/monthlyReportService.ts`:

```typescript
import { getEligibleProUsers, getUserByShop, markReportSent, EligibleUser } from "./eligibility";
import { fetchMonthlyReportPayload } from "./monthlyReportData";
import { getLastMonthRange } from "./dateRange";
import { renderMonthlyReportHtml } from "../../email/renderMonthlyReportHtml";
import { sendMonthlyReport } from "../mail/mailService";

const BATCH_SIZE = 10;
const MAX_RETRIES = 3;

export interface ReportJobOptions {
  dryRun?: boolean;
  shop?: string;
}

export interface ReportJobResult {
  sent: number;
  failed: number;
  skipped: number;
  durationMs: number;
  errors: { shop: string; error: string }[];
}

async function processShopWithRetry(
  user: EligibleUser,
  startDate: string,
  endDate: string,
  periodLabel: string,
  dryRun: boolean
): Promise<void> {
  let lastErr: Error | undefined;

  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    try {
      const payload = await fetchMonthlyReportPayload(user.myshopify_domain, startDate, endDate, periodLabel);
      const html = renderMonthlyReportHtml(payload, user);

      if (dryRun) return;

      // Send email first. If this succeeds, the email is delivered regardless of what happens next.
      await sendMonthlyReport(user, html, periodLabel);

      // Mark sent in a separate try/catch — a DB failure here must NOT trigger a retry
      // that would re-send the email. Worst case: merchant receives one extra email next
      // month's cron if lastMonthlyReportSentAt was never written. That is acceptable.
      try {
        await markReportSent(user.myshopify_domain);
      } catch (markErr: any) {
        console.error(`[MonthlyReport] WARN: email sent to ${user.myshopify_domain} but failed to update lastMonthlyReportSentAt: ${markErr.message}`);
      }
      return;
    } catch (err: any) {
      lastErr = err;
      console.warn(`[MonthlyReport] Attempt ${attempt}/${MAX_RETRIES} failed for ${user.myshopify_domain}: ${err.message}`);
      if (attempt < MAX_RETRIES) {
        await new Promise((r) => setTimeout(r, 1000 * attempt)); // exponential backoff: 1s, 2s
      }
    }
  }

  throw lastErr;
}

export async function runMonthlyReportJob(opts: ReportJobOptions = {}): Promise<ReportJobResult> {
  const { dryRun = false, shop } = opts;
  const startTime = Date.now();
  const { startDate, endDate, periodLabel } = getLastMonthRange();

  const users = shop ? await getUserByShop(shop) : await getEligibleProUsers();

  const result: ReportJobResult = { sent: 0, failed: 0, skipped: 0, durationMs: 0, errors: [] };

  for (let i = 0; i < users.length; i += BATCH_SIZE) {
    const batch = users.slice(i, i + BATCH_SIZE);

    await Promise.allSettled(
      batch.map(async (user) => {
        try {
          await processShopWithRetry(user, startDate, endDate, periodLabel, dryRun);
          if (dryRun) {
            result.skipped++;
          } else {
            result.sent++;
          }
        } catch (err: any) {
          result.failed++;
          result.errors.push({ shop: user.myshopify_domain, error: err.message });
          console.error(`[MonthlyReport] Failed for ${user.myshopify_domain}:`, err.message);
        }
      })
    );
  }

  result.durationMs = Date.now() - startTime;
  console.log(`[MonthlyReport] Job complete: ${JSON.stringify(result)}`);
  return result;
}
```

- [ ] **Step 10.2: Build check**

```bash
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 10.3: Commit**

```bash
git add src/services/report/monthlyReportService.ts
git commit -m "feat: add monthly report service orchestrator with batching and retries"
```

---

## Task 11: HTTP Controller + Route

**Files:**
- Create: `src/controllers/monthlyReportController.ts`
- Modify: `src/routes/analyticsRoutes.ts`

- [ ] **Step 11.1: Create `monthlyReportController.ts`**

Create `src/controllers/monthlyReportController.ts`:

```typescript
import { Request, Response } from "express";
import { runMonthlyReportJob } from "../services/report/monthlyReportService";

export async function runMonthlyReport(req: Request, res: Response): Promise<void> {
  try {
    const { dryRun, shop } = req.body ?? {};

    const result = await runMonthlyReportJob({
      dryRun: Boolean(dryRun),
      shop: typeof shop === "string" ? shop : undefined,
    });

    res.status(200).json(result);
  } catch (error: any) {
    console.error("[MonthlyReport] Controller error:", error);
    res.status(500).json({ error: "Monthly report job failed", detail: error.message });
  }
}
```

- [ ] **Step 11.2: Register route in `analyticsRoutes.ts`**

Add to `src/routes/analyticsRoutes.ts` (after existing imports, before existing `router.post` lines):

```typescript
import { cronAuth } from "../middleware/cronAuth";
import { runMonthlyReport } from "../controllers/monthlyReportController";
```

And add the route (after the existing `router.get("/suggestions", ...)` line):

```typescript
router.post("/cron/monthly-report", cronAuth, runMonthlyReport);
```

- [ ] **Step 11.3: Build check**

```bash
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 11.4: Commit**

```bash
git add src/controllers/monthlyReportController.ts src/routes/analyticsRoutes.ts
git commit -m "feat: add POST /api/analytics/cron/monthly-report endpoint"
```

---

## Task 12: Environment Variables

**Files:**
- Modify: `.env.example`

- [ ] **Step 12.1: Add new env vars to `.env.example`**

Append to `.env.example`:

```env
# Monthly Report (Cron)
CRON_SECRET=                          # Shared with Google Cloud Scheduler; required
BREVO_API_KEY=                        # Brevo transactional email API key
SHOPIFY_APP_URL=https://apps.shopify.com/bucks  # Base URL for email CTA links
MONTHLY_REPORT_FROM_EMAIL=reports@bucks.com
MONTHLY_REPORT_FROM_NAME=BUCKS
```

- [ ] **Step 12.2: Commit**

```bash
git add .env.example
git commit -m "chore: document monthly report env vars in .env.example"
```

---

## Task 13: Full Integration Test (dryRun)

This task verifies the full pipeline end-to-end using `dryRun: true` against a real shop in your dev environment (no email sent, no DB updated).

- [ ] **Step 13.1: Set env vars in your local `.env`**

Ensure these are set in `.env`:
```
CRON_SECRET=dev-test-secret
SHOPIFY_APP_URL=https://apps.shopify.com/bucks
MONTHLY_REPORT_FROM_EMAIL=reports@bucks.com
MONTHLY_REPORT_FROM_NAME=BUCKS
```
`BREVO_API_KEY` is NOT required for dryRun.

- [ ] **Step 13.2: Start the dev server**

```bash
npm run dev
```

- [ ] **Step 13.3: Trigger a dryRun for a single shop**

Replace `<SHOP>` with a shop domain that exists in your MongoDB `users` collection:

```bash
curl -X POST http://localhost:3001/api/analytics/cron/monthly-report \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dev-test-secret" \
  -d '{"dryRun": true, "shop": "<SHOP>.myshopify.com"}'
```

Expected response:
```json
{ "sent": 0, "failed": 0, "skipped": 1, "durationMs": <ms>, "errors": [] }
```

- [ ] **Step 13.4: Test unauthorized request**

```bash
curl -X POST http://localhost:3001/api/analytics/cron/monthly-report \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer wrong-secret" \
  -d '{}'
```

Expected: `401 Unauthorized`

- [ ] **Step 13.5: Run full test suite**

```bash
npm test
```

Expected: all tests pass (no regressions on existing routes).

- [ ] **Step 13.6: Final commit**

```bash
git add .
git commit -m "feat: monthly analytics report feature complete (phases 1-3)"
```

---

## Task 14: HTML Email Preview (Optional Dev Helper)

This is a developer convenience — not shipped as a permanent endpoint. To visually inspect the rendered email HTML during development, temporarily add a preview route that returns HTML directly in the browser.

- [ ] **Step 14.1: Test email render locally**

Add a temporary script `scripts/previewEmail.ts` (do NOT commit this to main):

```typescript
import dotenv from "dotenv";
dotenv.config();
import { fetchMonthlyReportPayload } from "../src/services/report/monthlyReportData";
import { getLastMonthRange } from "../src/services/report/dateRange";
import { renderMonthlyReportHtml } from "../src/email/renderMonthlyReportHtml";
import { getUserByShop } from "../src/services/report/eligibility";
import * as fs from "fs";

const SHOP = process.env.PREVIEW_SHOP || "buckscc-demo.myshopify.com";

(async () => {
  const { startDate, endDate, periodLabel } = getLastMonthRange();
  const [user] = await getUserByShop(SHOP);
  if (!user) { console.error("Shop not found:", SHOP); process.exit(1); }
  const payload = await fetchMonthlyReportPayload(SHOP, startDate, endDate, periodLabel);
  const html = renderMonthlyReportHtml(payload, user);
  fs.writeFileSync("preview-email.html", html);
  console.log("Written to preview-email.html — open in browser to inspect.");
  process.exit(0);
})();
```

Run with:
```bash
PREVIEW_SHOP=yourshop.myshopify.com npx ts-node --transpile-only scripts/previewEmail.ts
```

Open `preview-email.html` in Chrome to inspect the rendered email visually before sending to Brevo.

---

## Task 15: Figma Design Integration

> **Note:** Complete this task AFTER getting the Figma design from the user. The email HTML partials (Task 6–7) use the RFC spec as a baseline. Once you have the Figma design via Figma MCP, update the partials to match exactly.

- [ ] **Step 15.1:** User shares Figma design link or selects node in Figma desktop app.
- [ ] **Step 15.2:** Use `mcp0_get_design_context` or `mcp0_get_screenshot` to extract colors, spacing, font sizes, and layout.
- [ ] **Step 15.3:** Update email partials in `src/email/partials/` to match Figma spec (colors, padding, card layout).
- [ ] **Step 15.4:** Re-run `renderMonthlyReportHtml.test.ts` to confirm tests still pass.
- [ ] **Step 15.5:** Run `scripts/previewEmail.ts` to visually verify email against Figma design.
- [ ] **Step 15.6:** Commit: `git commit -m "feat: update email HTML to match Figma design"`

---

## Post-Implementation Checklist

- [ ] `POST /cron/monthly-report` returns 401 for wrong secret
- [ ] `dryRun: true` + `shop` param returns `{ skipped: 1, sent: 0, failed: 0 }`
- [ ] Stats numbers in rendered email match `/api/analytics/summary?shop=X` for same date range
- [ ] Email renders correctly in Gmail (web + mobile) — use [Litmus](https://litmus.com) or [Mail Tester](https://www.mail-tester.com)
- [ ] `lastMonthlyReportSentAt` is set after a real send
- [ ] Running the cron a second time the same month skips already-sent shops (`skipped > 0`, `sent = 0`)
- [ ] A shop with zero analytics data returns valid empty-state email (no crashes)

---

## Future: Phase 7 — Non-Pro One-Time Report

When ready, refer to RFC section "Future Enhancement: One-Time Report for Non-Pro Merchants". The infrastructure built here (email partials, mail service, cron auth) is fully reusable. New additions needed:
- `POST /api/analytics/cron/one-time-preview-report` endpoint
- `fetchPreviewReportPayload` (summary only)
- `renderPreviewReportHtml` (stats cards + upgrade CTA only)
- Eligibility query using `oneTimeAnalyticsReportSentAt: null` and `bucks_plan NOT IN PRO_ANALYTICS_PLANS`
