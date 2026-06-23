# Monthly Analytics Report — Design Spec

**Date:** 2026-06-23  
**Author:** BUCKS Engineering  
**Status:** Approved  
**RFC:** `docs/monthly-analytics-report-rfc.md`  
**Implementation Plan:** `docs/superpowers/plans/2026-06-23-monthly-analytics-report.md`

---

## 1. Problem

Pro merchants must manually log into BUCKS to review performance. There is no automated periodic summary of currency conversion activity, sales by market, or actionable recommendations. This means:

- Merchants forget to check analytics between sessions.
- No monthly value-reinforcement touchpoint for the Pro plan.
- Missed trends (top currencies, country performance) go unnoticed.

---

## 2. Solution

A **monthly automated email report** generated and delivered entirely from the analytics Express backend. Cloud Scheduler fires a secured HTTP POST on the 1st of each month at 09:00 UTC. The backend fetches analytics data for each eligible Pro merchant, renders an email-safe HTML report, and sends it via Brevo.

**Scope — v1:** Pro plan merchants only (recurring monthly).  
**Scope — Phase 7 (future):** One-time preview report for non-Pro merchants.

---

## 3. Architecture

### 3.1 System Components

```
Google Cloud Scheduler (0 9 1 * *)
    │  POST /api/analytics/cron/monthly-report
    │  Authorization: Bearer <CRON_SECRET>
    ▼
Analytics Express Backend
    ├── cronAuth middleware         — validates CRON_SECRET (timing-safe)
    ├── monthlyReportController     — HTTP handler, parses dryRun/shop params
    ├── monthlyReportService        — batch orchestrator (10 shops/batch, 3 retries)
    │   ├── eligibility             — MongoDB: getEligibleProUsers / markReportSent
    │   ├── fetchMonthlyReportPayload — 5 parallel queries (analyticsService + suggestionsService)
    │   ├── renderMonthlyReportHtml — assembles HTML from 5 partials
    │   └── mailService             — Brevo REST API send
    ▼
MongoDB (users)           Brevo (merchant inbox)
```

### 3.2 Import Dependency Graph

```
analyticsRoutes.ts
    └── cronAuth (middleware/cronAuth.ts)
    └── monthlyReportController.ts
            └── monthlyReportService.ts
                    ├── eligibility.ts          → prisma (utils/prisma.ts)
                    ├── dateRange.ts            (pure, no deps)
                    ├── monthlyReportData.ts
                    │       ├── analyticsService.ts  → clickhouse, currencyConverter, prisma
                    │       └── suggestionsService   (existing, unchanged)
                    ├── renderMonthlyReportHtml.ts
                    │       └── email/partials/* (pure string builders, no deps)
                    └── mailService.ts          → fetch (Brevo REST)
```

**No circular imports.** All data flows in one direction: route → controller → service → data/render/mail.

### 3.3 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Data fetch | Internal service calls (not self-HTTP) | Avoids network round-trip per shop; analytics data already on same process |
| Email rendering | Hand-built email-safe HTML (inline styles, table layout) | No React/Polaris/Recharts in email clients; reliable across Gmail + Outlook |
| Scheduler trigger | Google Cloud Scheduler → Bearer token | Existing team pattern; no new infra |
| Testing API | Same endpoint with `dryRun: true` + `shop` param | No separate test endpoint needed; CRON_SECRET required even for dry runs |
| Batching | 10 shops per batch, `Promise.allSettled` | One failing shop cannot block others; parallel within batch |
| Retries | 3 attempts per shop | Transient ClickHouse/Brevo failures handled without full job restart |
| Idempotency | `lastMonthlyReportSentAt` < start of current month | Duplicate cron fires (e.g. Scheduler retry) skip already-sent shops |

---

## 4. Data Model Changes

**MongoDB `users` collection** — 3 new optional fields added to `prisma/schema.prisma`:

```prisma
lastMonthlyReportSentAt      DateTime?             // Pro monthly idempotency
oneTimeAnalyticsReportSentAt DateTime?             // Non-Pro one-time (Phase 7)
monthlyReportOptOut          Boolean? @default(false)  // Future opt-out toggle
```

MongoDB is schema-flexible — no migration file required. Existing documents without these fields return `null`, which Prisma `DateTime?` / `Boolean?` handles correctly.

---

## 5. API Contract

### Cron Endpoint

```
POST /api/analytics/cron/monthly-report
Authorization: Bearer <CRON_SECRET>
Content-Type: application/json

Body (all optional):
{
  "dryRun": true,                        // render but do not send, do not update DB
  "shop": "example.myshopify.com"        // process single shop only (for testing)
}

Success response 200:
{
  "sent": 3,
  "failed": 0,
  "skipped": 0,
  "durationMs": 4821,
  "errors": []
}

Error responses:
  401  — missing or wrong CRON_SECRET
  500  — CRON_SECRET env var not configured
```

### dryRun behaviour

- Fetches analytics data and renders HTML for each shop.
- Does **not** call Brevo.
- Does **not** update `lastMonthlyReportSentAt`.
- Returns `{ sent: 0, skipped: N, failed: 0 }`.

---

## 6. Eligibility Query

Pro merchants are eligible when **all** of the following are true:

| Condition | Field |
|-----------|-------|
| On a Pro analytics plan | `bucks_plan IN ['bucks_premium_pro', 'bucks_premium_pro_annual_60', 'bucks_premium_pro_annual_65']` |
| App is installed | `is_active = true` |
| Has not opted out | `monthlyReportOptOut != true` |
| Has an email address | `email != ""` OR `customer_email != ""` |
| Has not been sent this month | `lastMonthlyReportSentAt IS NULL` OR `lastMonthlyReportSentAt < start of current month (UTC)` |

---

## 7. Analytics Data Fetched Per Shop

All 5 queries run in parallel for the **previous full calendar month (UTC)**:

| Data | Source | Used in email section |
|------|--------|-----------------------|
| `summary` | `analyticsService.getSummaryData()` | Stat cards |
| `trends` | `analyticsService.getTrendsData()` | Conversion trends bar chart |
| `revenue` | `analyticsService.getRevenueByCurrencyData()` | Revenue by currency table |
| `countries` | `analyticsService.getSalesByCountryData()` | Top countries table |
| `suggestions` | `suggestionsService.getSuggestionsForShop()` | Recommendations section |

Date range: `startDate = YYYY-MM-01`, `endDate = YYYY-MM-<last_day>` of previous month, computed in UTC.

---

## 8. Email Design Spec

### 8.1 Layout Structure

```
┌─────────────────────────────────────────┐  max-width: 600px
│  BUCKS (white on #008060)               │  Header
│  "Your May 2026 Analytics Report"       │
├─────────────────────────────────────────┤
│  Hi {first_name},                       │  Greeting
│  Here's how {shop_name} performed...    │
├─────────────────────────────────────────┤
│  [Total Visits] [Currency Clicks]       │  Stat Cards (2×2 table)
│  [Total Sales]  [Top Currency]          │
├─────────────────────────────────────────┤
│  Currency Conversion Trends             │  Horizontal bar chart (top 5)
│  EUR ████████████ 1,200                 │
│  USD ██████       600                   │
├─────────────────────────────────────────┤
│  Revenue by Currency                    │  % bar + amount table (top 5)
│  EUR  ██████████ 60%  3,200.00 USD      │
├─────────────────────────────────────────┤
│  Top Sales by Country                   │  Table (top 5)
│  Germany     2,000.00 USD               │
├─────────────────────────────────────────┤
│  Smart Recommendations (0–3 items)      │  Green left-border cards
│  ▌ Add EUR to your currencies           │
├─────────────────────────────────────────┤
│  [ View full analytics → ]              │  CTA button (#008060)
├─────────────────────────────────────────┤
│  Footer: unsubscribe note + copyright   │
└─────────────────────────────────────────┘
```

### 8.2 Visual Tokens

| Token | Value |
|-------|-------|
| Max width | 600px |
| Primary colour | `#008060` (BUCKS green) |
| Body background | `#f4f4f4` |
| Card background | `#f9f9f9` |
| Card border | `1px solid #e8e8e8` |
| Card border-radius | `12px` |
| Section title | `16px, font-weight: 700, #303030` |
| Stat card label | `11px, #303030A6, uppercase` |
| Stat card value | `22px, font-weight: 700, #303030` |
| Body text | `13–14px, #303030A6, line-height: 1.6` |
| CTA button | `#008060` bg, white text, `15px bold`, `8px` border-radius, `14px 32px` padding |
| Recommendation card | `#f0faf6` bg, `4px solid #008060` left border |
| Font stack | `Arial, sans-serif` (no web fonts — email client safe) |

> **Note:** These tokens are the RFC baseline. They will be updated in Task 15 of the implementation plan once the Figma design is shared via Figma MCP.

### 8.3 Email Client Compatibility Rules

- **Table-based layout only** — no CSS Grid, no Flexbox.
- **Inline styles only** — no `<style>` blocks with class selectors (Outlook strips them).
- `bgcolor` attribute on `<table>` / `<td>` for background colours (Outlook fallback).
- `border-radius` on containers: renders in Gmail, degrades gracefully in Outlook (square corners).
- No external fonts (`@font-face` not supported in most clients).
- Bar charts: HTML `<table>` width-percentage columns coloured with `bgcolor` — no `<canvas>`, no SVG.
- All images must have `alt` text and explicit `width`/`height`.

### 8.4 Empty State

| Section | Empty behaviour |
|---------|----------------|
| Stat cards | Always rendered — shows `0` values |
| Conversion trends | Section hidden if `trends` array is empty |
| Revenue by currency | Section hidden if `revenue` array is empty |
| Top countries | Section hidden if `countries` array is empty |
| Recommendations | Section hidden if `suggestions.suggestions` is empty |

---

## 9. New Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `CRON_SECRET` | Yes | Shared with Cloud Scheduler; used for `Authorization: Bearer` header validation |
| `BREVO_API_KEY` | Yes (for real sends) | Brevo transactional API key |
| `SHOPIFY_APP_URL` | Yes | Base URL for email CTA links (e.g. `https://apps.shopify.com/bucks`) |
| `MONTHLY_REPORT_FROM_EMAIL` | Yes | Sender email address (e.g. `reports@bucks.com`) |
| `MONTHLY_REPORT_FROM_NAME` | Yes | Sender display name (e.g. `BUCKS`) |

---

## 10. Security

- `POST`-only cron endpoint — `GET /cron/monthly-report` returns 404.
- `CRON_SECRET` compared with `crypto.timingSafeEqual` — prevents timing attacks.
- No shop data in HTTP response — only counts (`sent`, `failed`, `skipped`) and shop domains in server-side error logs.
- `dryRun` mode requires the same `CRON_SECRET` — not an open preview endpoint.

---

## 11. Error Handling & Monitoring

| Scenario | Handling |
|----------|----------|
| Missing email on user | Skipped at eligibility query (excluded by filter) |
| Analytics query fails for shop | Retry 3× then log + count as `failed`; continue batch |
| Brevo returns non-2xx | Throw error → retry → mark `failed`; `lastMonthlyReportSentAt` NOT updated |
| Duplicate cron trigger same month | `lastMonthlyReportSentAt` check skips already-sent shops |
| Zero analytics data for shop | Valid empty-state email sent (stat cards show 0, sections with no data hidden) |
| `GOOGLE_SERVICE_ACCOUNT_BASE64` unset | Suggestions AI call fails silently; `suggestionsService` falls back to manual suggestions |
| `CRON_SECRET` not set in env | Endpoint returns 500 immediately |

**Job summary log** on every run:
```
[MonthlyReport] Job complete: { sent: 12, failed: 1, skipped: 0, durationMs: 8432 }
```

---

## 12. Testing Strategy

| Test | Type | Location |
|------|------|----------|
| `getLastMonthRange()` — date boundary cases | Unit | `src/services/report/dateRange.test.ts` |
| `getEligibleProUsers()` / `getUserByShop()` / `markReportSent()` — Prisma mocked | Unit | `src/services/report/eligibility.test.ts` |
| `renderMonthlyReportHtml()` — HTML output assertions | Unit | `src/email/renderMonthlyReportHtml.test.ts` |
| `sendMonthlyReport()` — Brevo fetch mocked | Unit | `src/services/mail/mailService.test.ts` |
| `cronAuth` — correct/wrong/missing secret | Unit | `src/middleware/cronAuth.test.ts` |
| Full pipeline dryRun via curl | Integration | Manual (see plan Task 13) |
| Gmail + Outlook rendering | Visual QA | `scripts/previewEmail.ts` → open in browser / Litmus |

---

## 13. File Structure (New Files Only)

```
src/
├── controllers/
│   └── monthlyReportController.ts       NEW
├── middleware/
│   └── cronAuth.ts                      NEW
├── services/
│   ├── analytics/
│   │   └── analyticsService.ts          NEW  (extracted from controllers)
│   ├── mail/
│   │   └── mailService.ts               NEW
│   └── report/
│       ├── dateRange.ts                 NEW
│       ├── eligibility.ts               NEW
│       ├── monthlyReportData.ts         NEW
│       └── monthlyReportService.ts      NEW
└── email/
    ├── renderMonthlyReportHtml.ts       NEW
    └── partials/
        ├── statsCardsHtml.ts            NEW
        ├── conversionTrendsHtml.ts      NEW
        ├── revenueByCurrencyHtml.ts     NEW
        ├── performanceTablesHtml.ts     NEW
        └── recommendationsHtml.ts      NEW
```

---

## 14. Out of Scope (v1)

- Frontend changes in the main Next.js app.
- Merchant opt-out toggle in app settings (field added to schema for future use).
- UTM tracking on CTA links.
- Non-Pro one-time preview report (Phase 7 — future).
- Cloud Scheduler configuration (infrastructure task, separate from code).
- Slack webhook on job completion.

---

## 15. Future: Phase 7 — Non-Pro One-Time Report

Reuses all v1 infrastructure. Additions needed:

- `POST /api/analytics/cron/one-time-preview-report` endpoint
- `fetchPreviewReportPayload()` — summary data only
- `renderPreviewReportHtml()` — stat cards + upgrade CTA only (no charts/tables)
- Eligibility: `bucks_plan NOT IN PRO_PLANS` + `oneTimeAnalyticsReportSentAt = null`
- CTA: "Upgrade to Pro analytics →" → `/pricing?shop=...`
