# RFC: Monthly Analytics Report for Pro Merchants

**Metadata:**

- **Author:** BUCKS Engineering
- **Status:** Draft
- **Reviewers:** CPO, Product Owner, Analytics Team
- **Type:** #feature-rfc

---

## 1. Problem

Pro merchants with full analytics access must **manually log into BUCKS** to review store performance. There is no automated way to deliver a periodic summary of currency conversion activity, sales by market, or actionable recommendations.

- **Merchant pain points:**
  - Performance insights are only available inside the app; merchants forget to check.
  - No monthly touchpoint reinforcing the value of the Pro analytics plan.
  - Merchants who are busy running their store miss trends (top currencies, country performance, low-conversion markets).
- **Current workarounds:**
  - Merchants open `/analytics` and change the date filter to "Last month" themselves.
  - Support cannot proactively share performance summaries.
- **Competitor context:**
  - Many SaaS analytics products (Klaviyo, Google Analytics email summaries, Shopify reports) send periodic email digests. Merchants expect a monthly "here's how you did" email for paid analytics tiers.

---

## 2. Why This Matters?

- **Why now:**
  - Full analytics dashboard is live for Pro plan merchants (`bucks_premium_pro*`).
  - Analytics backend (Express) already aggregates all required data via existing endpoints.
  - Shared MongoDB gives the analytics backend access to merchant email and plan eligibility.
  - Team already uses **Google Cloud Scheduler** for scheduled jobs — no new scheduling infra required.
- **Benefits:**
  - Increases perceived value of Pro analytics subscription.
  - Drives re-engagement with the analytics dashboard (email CTA → in-app analytics).
  - Surfaces `suggestions` proactively without merchants opening the app.
  - Reduces "I didn't know my store was performing well in EUR" support conversations.
  - **(Future)** One-time non-Pro preview report creates an upgrade funnel to Pro analytics.
- **Cost of inaction:**
  - Pro analytics remains a passive feature merchants must remember to use.
  - Lower differentiation vs competitors offering automated reporting.
  - Missed opportunity for retention and upgrade justification.
- **Non-Pro merchants (future):**
  - Free and non-Pro paid merchants have limited analytics (summary only) and never receive any automated performance summary.
  - No touchpoint to demonstrate the value of upgrading to Pro analytics.

---

## 3. Proposed Solution

Build a **monthly automated email report** generated and sent entirely from the **analytics Express backend**. The main Next.js app is not involved in report generation or delivery.

**Scope:**
- **v1:** Monthly recurring report for **Pro analytics merchants only**.
- **Future (Phase 7):** One-time preview report for **non-Pro merchants** — sent once per shop to showcase analytics value and drive Pro upgrades. Reuses the same HTML pipeline with a reduced content tier.

### Approach summary

| Concern | Owner |
|---------|--------|
| Cron trigger | Google Cloud Scheduler → HTTP POST to analytics backend |
| Eligibility (Pro plan, email, active) | Analytics backend (query shared MongoDB `users`) |
| Analytics data | Analytics backend (internal service calls — not self-HTTP) |
| HTML email rendering | Analytics backend (email-safe HTML) |
| Email delivery | Analytics backend (Brevo) |
| Deep link to full dashboard | Main app URL in CTA button only |

### Why analytics backend (not main app)?

- Analytics data already lives here; internal calls avoid HTTP round-trips per shop.
- `suggestions` controller already exists on the same server.
- Shared MongoDB provides `email`, `bucks_plan`, `shop_owner`, `currency` without cross-service calls.
- Monthly batch job is naturally co-located with data aggregation.

### Why HTML-only email (primary)?

- Email clients do not support React, Polaris, or Recharts.
- Hand-built email HTML can closely match the analytics dashboard design for stats cards, tables, and bar-style chart visualizations.
- Reliable across Gmail, Outlook, and mobile clients.
- No Puppeteer/headless browser dependency in production.

### What existing systems are reused?

- Analytics Express routes/controllers: `getSummary`, `getTrends`, `getRevenueByCurrency`, `getSalesByCountry`, `getSuggestions`
- Shared MongoDB `users` collection (plan, email, shop domain)
- `calculateDateRange("last_month")` logic (ported or duplicated in analytics backend)
- Brevo API pattern (same as main app `mailEngine.js`)
- Google Cloud Scheduler (existing team pattern)

### What new systems are introduced?

- `POST /api/analytics/cron/monthly-report` — secured cron endpoint on analytics backend
- `monthlyReportService` — orchestrates fetch → render → send per shop
- `renderMonthlyReportHtml` — email HTML template builders
- `mailService` — Brevo send wrapper on analytics backend
- `lastMonthlyReportSentAt` field on `users` (idempotency for Pro monthly sends)
- `oneTimeAnalyticsReportSentAt` field on `users` (idempotency for non-Pro one-time sends — **future**)

### Architecture diagram

```
┌──────────────────────────┐
│  Google Cloud Scheduler   │
│  Schedule: 0 9 1 * *      │
└────────────┬─────────────┘
             │ POST /api/analytics/cron/monthly-report
             │ Authorization: Bearer CRON_SECRET
             ▼
┌──────────────────────────────────────────────────────────┐
│  Analytics Express Backend                                │
│                                                           │
│  1. Verify CRON_SECRET                                    │
│  2. Query MongoDB users (Pro plan + active + email)       │
│  3. For each shop (batched):                              │
│     a. Internal: getSummary, getTrends, getRevenue...     │
│     b. Internal: getSuggestions                           │
│     c. renderMonthlyReportHtml()                          │
│     d. send via Brevo                                     │
│     e. Update lastMonthlyReportSentAt                     │
└──────────────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────┐     ┌──────────────────────────┐
│  MongoDB (shared)         │     │  Brevo (transactional)    │
│  users collection         │     │  merchant inbox           │
└──────────────────────────┘     └──────────────────────────┘
```

### Edge cases & limitations

| Case | Handling |
|------|----------|
| No analytics activity in period | Send report with zeros + encouragement copy |
| Missing email on user | Skip shop, log |
| User not on Pro plan | Excluded by query filter |
| User uninstalled (`is_active: false`) | Excluded by query filter |
| Analytics query fails for one shop | Retry 3×, skip, continue batch, log error |
| Duplicate cron trigger | `lastMonthlyReportSentAt` idempotency check |
| Large merchant count | Process in batches; return job summary JSON |
| Charts in email | HTML bar/table visualizations — not interactive Recharts |
| Outlook CSS limitations | Table-based layout, inline styles only |

### Migrations from existing

- No changes to existing analytics tracking endpoints (`/track-visit`, `/order`, etc.).
- No changes to main app analytics proxy or dashboard UI.
- Main app only needs `SHOPIFY_APP_URL` referenced in email CTA link (env var on analytics backend).

---

## 4. End-to-End Flow

### Merchant flow (admin)

1. Merchant is on Pro analytics plan (`bucks_premium_pro`, `bucks_premium_pro_annual_60`, or `bucks_premium_pro_annual_65`).
2. On the **1st of each month**, merchant receives email: **"Your BUCKS Analytics Report — [Month Year]"**.
3. Email contains:
   - Greeting with shop owner name
   - 4 stat cards (visits, currency clicks, total sales, top currency)
   - Currency conversion trends (HTML horizontal bars, top 5)
   - Revenue by currency breakdown (HTML table with % bars)
   - Top sales by currency and country tables
   - Smart recommendations (from `suggestions` API)
   - **"View full analytics"** button → main app `/analytics?shop=...`
4. Merchant takes no action to opt in for v1 (automatic for eligible Pro users).

### Customer flow (storefront)

No storefront impact. Report is merchant-facing only.

### Technical flow

```
Day 1 of month, 09:00 UTC
  → Cloud Scheduler fires POST to analytics backend
  → Endpoint validates Bearer CRON_SECRET
  → Query: users where bucks_plan IN PRO_ANALYTICS_PLANS
           AND is_active = true
           AND (email OR customer_email) IS NOT empty
           AND lastMonthlyReportSentAt < start of current month
  → For each shop (batch of 10):
       startDate/endDate = previous calendar month (UTC)
       summary    = getSummary(shop, dates)      // internal
       trends     = getTrends(shop, dates)       // internal
       revenue    = getRevenueByCurrency(...)    // internal
       countries  = getSalesByCountry(...)       // internal
       suggestions = getSuggestions(shop, ...)   // internal
       html = renderMonthlyReportHtml(payload)
       brevo.send({ to: email, html, subject })
       users.update({ lastMonthlyReportSentAt: now })
  → Return { sent, failed, skipped, duration }
```

### Authentication flow (Cloud Scheduler → API)

```
Cloud Scheduler job:
  Method:  POST
  URL:     https://analytics.bucks.helixo.co/api/analytics/cron/monthly-report
  Header:  Authorization: Bearer <CRON_SECRET>
  Body:    {} (optional: { "dryRun": true, "shop": "test.myshopify.com" })

Analytics endpoint:
  if Authorization !== Bearer CRON_SECRET → 401
  if method !== POST → 405
  else → run job
```

`CRON_SECRET` stored in:
- GCP Secret Manager (Scheduler job header)
- Analytics backend environment variable

### Email content structure (HTML)

```
┌─────────────────────────────────────────┐
│  BUCKS logo + "Your May 2026 Report"    │
├─────────────────────────────────────────┤
│  Hi {shop_owner},                       │
│  Here's how {shop} performed last month │
├─────────────────────────────────────────┤
│  [Visits] [Clicks] [Sales] [Top CCY]   │  ← 2×2 table cards
├─────────────────────────────────────────┤
│  Currency Conversion Trends             │  ← HTML horizontal bars
├─────────────────────────────────────────┤
│  Revenue by Currency                    │  ← HTML % breakdown table
├─────────────────────────────────────────┤
│  Sales by Currency | Sales by Country   │  ← HTML tables (top 5)
├─────────────────────────────────────────┤
│  Recommendations (1–3 items)            │  ← title + description
├─────────────────────────────────────────┤
│  [ View full analytics → ]              │  ← CTA to main app
└─────────────────────────────────────────┘
```

---

## 5. Implementation Details

### Data model changes

**MongoDB `users` collection** (shared, via Prisma on main app — analytics backend reads/writes same field):

```prisma
// prisma/schema.prisma — add to users model
lastMonthlyReportSentAt       DateTime?   // Pro monthly report idempotency
oneTimeAnalyticsReportSentAt  DateTime?   // Non-Pro one-time report idempotency (future)
monthlyReportOptOut           Boolean?  @default(false)  // optional v2
```

Analytics backend uses native MongoDB driver or existing DB client to read/write this field.

### Backend changes (Analytics Express)

#### New files

| File | Responsibility |
|------|----------------|
| `services/monthlyReportService.js` | Orchestrate per-shop report: fetch → render → send |
| `services/monthlyReportData.js` | Call internal controllers/services for all endpoints |
| `email/renderMonthlyReportHtml.js` | Compose full HTML email |
| `email/partials/statsCardsHtml.js` | 4 stat cards (table layout) |
| `email/partials/conversionTrendsHtml.js` | HTML horizontal bar chart |
| `email/partials/revenueByCurrencyHtml.js` | % breakdown table |
| `email/partials/performanceTablesHtml.js` | Top 5 currency + country tables |
| `email/partials/recommendationsHtml.js` | Suggestions section |
| `services/mailService.js` | Brevo send wrapper |
| `middleware/cronAuth.js` | Verify `CRON_SECRET` |
| `controllers/monthlyReportController.js` | HTTP handler for cron endpoint |

#### New route

Add to `analyticsRoutes`:

```ts
import { cronAuth } from "../middleware/cronAuth";
import { runMonthlyReport } from "../controllers/monthlyReportController";

router.post("/cron/monthly-report", cronAuth, runMonthlyReport);
```

Full path: `POST /api/analytics/cron/monthly-report`

#### Internal data fetch (do NOT self-HTTP)

```ts
// monthlyReportData.ts — call controller logic directly
import { getSummaryData } from "../controllers/analyticsFetchController";
// OR extract shared service layer from existing controllers

export async function fetchMonthlyReportPayload(shop: string) {
  const { startDate, endDate, periodLabel } = getLastMonthRange();

  const [summary, trends, revenue, countries, suggestions] = await Promise.all([
    analyticsService.getSummary(shop, startDate, endDate),
    analyticsService.getTrends(shop, startDate, endDate),
    analyticsService.getRevenueByCurrency(shop, startDate, endDate),
    analyticsService.getSalesByCountry(shop, startDate, endDate),
    suggestionsService.getSuggestions(shop, "last_month"),
  ]);

  return { shop, periodLabel, startDate, endDate, summary, trends, revenue, countries, suggestions };
}
```

Refactor existing `analyticsFetchController` functions into a shared `analyticsService` if they are currently tied to `req/res` — this is the main structural change on the analytics backend.

#### Eligibility query

```ts
const PRO_ANALYTICS_PLANS = [
  "bucks_premium_pro",
  "bucks_premium_pro_annual_60",
  "bucks_premium_pro_annual_65",
];

const users = await db.users.find({
  bucks_plan: { $in: PRO_ANALYTICS_PLANS },
  is_active: true,
  $or: [{ email: { $ne: "" } }, { customer_email: { $ne: "" } }],
  monthlyReportOptOut: { $ne: true },
  $or: [
    { lastMonthlyReportSentAt: null },
    { lastMonthlyReportSentAt: { $lt: startOfCurrentMonth } },
  ],
});
```

#### Cron controller

```ts
export async function runMonthlyReport(req: Request, res: Response) {
  const { dryRun, shop: singleShop } = req.body ?? {};

  const users = singleShop
    ? await getUserByShop(singleShop)
    : await getEligibleProUsers();

  const results = { sent: 0, failed: 0, skipped: 0, errors: [] };

  for (const user of users) {
    try {
      const payload = await fetchMonthlyReportPayload(user.myshopify_domain);
      const html = renderMonthlyReportHtml(payload, user);

      if (dryRun) {
        results.skipped++;
        continue; // or save HTML to /tmp for preview
      }

      await mailService.sendMonthlyReport(user, html, payload.periodLabel);
      await markReportSent(user.myshopify_domain);
      results.sent++;
    } catch (err) {
      results.failed++;
      results.errors.push({ shop: user.myshopify_domain, error: err.message });
    }
  }

  return res.status(200).json(results);
}
```

#### Environment variables (analytics backend)

```env
CRON_SECRET=                          # shared with Cloud Scheduler
BREVO_API_KEY=                        # transactional email
SHOPIFY_APP_URL=                      # CTA link base
MONTHLY_REPORT_FROM_EMAIL=reports@bucks.com
MONTHLY_REPORT_FROM_NAME=BUCKS
```

### Frontend changes (main app)

**None required for v1.**

Optional later:
- Settings toggle for `monthlyReportOptOut` on main app settings page
- Analytics page banner: "Your monthly report was sent on [date]"

### HTML email design spec

| Element | Value |
|---------|-------|
| Max width | 600px |
| Font | Inter, Arial, sans-serif |
| Card border-radius | 12px |
| Title text | 12px, `#303030A6` |
| Value text | 14px bold, `#303030` |
| CTA button | `#008060` background, white text |
| Bar chart colors | Match `getColorForCurrency()` palette |
| Section order | Stats → Trends → Revenue → Tables → Recommendations → CTA |

### Edge cases & special handling

| Scenario | Behavior |
|----------|----------|
| `dryRun: true` in request body | Render report, do not send email, do not update `lastMonthlyReportSentAt` |
| `shop` in request body | Process single shop only (for testing) |
| Zero data | Show empty-state copy matching in-app analytics |
| Suggestions empty | Hide recommendations section |
| Brevo failure | Log error, do not update `lastMonthlyReportSentAt`, count as failed |
| Scheduler timeout | Design endpoint to complete within Scheduler HTTP timeout, or process in chunks with `?offset=0&limit=50` and chain Scheduler jobs |
| Secret rotation | Update both GCP Secret Manager and analytics env; old secret invalid immediately |

### Security

- `POST` only on cron endpoint
- `CRON_SECRET` verified on every request (constant-time compare)
- No shop data exposed in cron response (only counts + shop domains in error log server-side)
- Email contains only that merchant's own data
- Preview/dry-run endpoint disabled in production unless `CRON_SECRET` provided

### Monitoring

- Log job summary: `{ sent, failed, skipped, durationMs }`
- Alert if `failed > 0` or `sent === 0` when eligible users exist
- Optional: Slack webhook on job completion (same pattern as main app install logs)

---

## 6. Open Questions

1. **Opt-out in v1?** Send automatically to all Pro users, or add Settings toggle before launch?
2. **Send time/timezone?** 09:00 UTC on the 1st — or adjust per merchant timezone later?
3. **Brevo template vs raw HTML?** Code-generated HTML (faster v1) vs designed Brevo template (easier marketing edits)?
4. **Empty month behavior?** Still send "quiet month" email, or skip merchants with zero visits?
5. **Scheduler timeout at scale?** How many Pro merchants today? Do we need chunked Scheduler jobs?
6. **Success metrics?** Track email open rate (Brevo), CTA clicks (UTM params), analytics page visits post-send?
7. **Non-Pro one-time trigger (future)?** Send after N days on free plan, after first analytics activity threshold, or as a one-time campaign? *(See Phase 7)*

---

## Implementation Plan

| Phase | Goal | Scope |
|-------|------|-------|
| **Phase 1** | Data layer + internal service refactor | Extract `analyticsService` from controllers; `fetchMonthlyReportPayload`; `getLastMonthRange` |
| **Phase 2** | HTML email renderer | All partials + `renderMonthlyReportHtml`; preview via `dryRun` endpoint |
| **Phase 3** | Send + cron | `mailService` (Brevo), `cronAuth`, `POST /cron/monthly-report`, `lastMonthlyReportSentAt` |
| **Phase 4** | Cloud Scheduler + staging test | Configure Scheduler job; test with 1 shop; test in Gmail + Outlook |
| **Phase 5** | Production rollout | Enable for all Pro merchants; monitor first run |
| **Phase 6 (optional)** | Merchant opt-out + metrics | Settings toggle, UTM tracking, Brevo template migration |
| **Phase 7 (future)** | One-time report for non-Pro merchants | Reduced-content preview email, upgrade CTA, send-once idempotency |

Each phase ships independently. Phase 1–3 can be tested locally without Scheduler.

---

## Future Enhancement: One-Time Report for Non-Pro Merchants (Phase 7)

After the Pro monthly report is stable, extend the same analytics backend pipeline to send a **single preview report** to non-Pro merchants. This is an upgrade funnel touchpoint — not a recurring monthly email.

### Goal

Give free and non-Pro paid merchants a **one-time taste** of analytics value, using data they already have access to (summary tier), and encourage upgrade to Pro for full charts, tables, and monthly reports.

### How it differs from Pro monthly report

| Aspect | Pro monthly report (v1) | Non-Pro one-time report (future) |
|--------|-------------------------|----------------------------------|
| **Frequency** | Every month (1st) | **Once per shop, ever** |
| **Eligibility** | `bucks_premium_pro*` | Active merchants **not** on Pro analytics plans |
| **Data fetched** | summary + trends + revenue + country + suggestions | **summary only** (matches in-app limited analytics) |
| **Email content** | Full report (cards, charts, tables, recommendations) | Stats cards + short insight copy + **upgrade CTA** |
| **Idempotency field** | `lastMonthlyReportSentAt` | `oneTimeAnalyticsReportSentAt` |
| **Trigger** | Cloud Scheduler monthly cron | TBD — event-based or one-time campaign job |
| **CTA** | "View full analytics" | "Upgrade to Pro analytics" → `/pricing?shop=...` |

### Proposed non-Pro email content

```
┌─────────────────────────────────────────┐
│  BUCKS logo + "Your Store Snapshot"     │
├─────────────────────────────────────────┤
│  Hi {shop_owner},                       │
│  Here's a snapshot of how {shop} is     │
│  performing with BUCKS                  │
├─────────────────────────────────────────┤
│  [Visits] [Clicks] [Sales] [Top CCY]   │  ← summary stats only
├─────────────────────────────────────────┤
│  "Unlock full analytics — see trends,   │
│   revenue by currency, country          │
│   breakdown, and smart recommendations" │
├─────────────────────────────────────────┤
│  [ Upgrade to Pro analytics → ]         │  ← pricing page
└─────────────────────────────────────────┘
```

No trends charts, performance tables, or suggestions in the non-Pro email — those are Pro-only features in the app today.

### Eligibility query (future)

```ts
const PRO_ANALYTICS_PLANS = [
  "bucks_premium_pro",
  "bucks_premium_pro_annual_60",
  "bucks_premium_pro_annual_65",
];

const users = await db.users.find({
  bucks_plan: { $nin: PRO_ANALYTICS_PLANS },
  is_active: true,
  oneTimeAnalyticsReportSentAt: null,  // never sent before
  $or: [{ email: { $ne: "" } }, { customer_email: { $ne: "" } }],
  // optional: minimum activity threshold
  // e.g. totalVisits > 0 in last 30 days
});
```

### Trigger options (to decide in Phase 7)

| Option | Description |
|--------|-------------|
| **A — Time-based** | Send once 30 days after install if still non-Pro |
| **B — Activity-based** | Send once when shop hits N visits or N currency clicks |
| **C — One-time campaign** | Manual Cloud Scheduler job / admin trigger for existing non-Pro base |
| **D — On analytics page first visit** | Queue send 24h after merchant first opens `/analytics` |

Recommendation: start with **Option C** (controlled rollout) before automating A or B.

### Backend changes (future, reuses v1 infrastructure)

| Component | Change |
|-----------|--------|
| `monthlyReportService` | Generalize to `reportService` with `reportType: 'monthly_pro' \| 'one_time_preview'` |
| `fetchMonthlyReportPayload` | Add `fetchPreviewReportPayload` — summary only |
| `renderMonthlyReportHtml` | Add `renderPreviewReportHtml` — reduced template + upgrade CTA |
| `POST /cron/monthly-report` | Unchanged (Pro only) |
| `POST /cron/one-time-preview-report` | **New** secured endpoint for non-Pro one-time sends |
| `users.oneTimeAnalyticsReportSentAt` | Set on successful send; never send again |

### Edge cases (non-Pro one-time)

| Case | Handling |
|------|----------|
| Merchant upgrades to Pro before send | Skip one-time preview; they receive Pro monthly report instead |
| Merchant already received preview | `oneTimeAnalyticsReportSentAt` set — never resend |
| Zero activity | Skip or send with empty-state + upgrade message (TBD) |
| Merchant uninstalls | `is_active: false` — exclude |

### Success metrics (future)

- One-time email open rate
- Upgrade CTA click-through to pricing
- Pro plan conversion rate within 30 days of preview send
- Compare: merchants who received preview vs control group

### Testing checklist

- [ ] `dryRun` + single `shop` returns expected HTML payload
- [ ] Stats numbers match dashboard for same shop + `last_month`
- [ ] Email renders correctly in Gmail (web + mobile)
- [ ] Email renders correctly in Outlook
- [ ] `CRON_SECRET` rejection returns 401
- [ ] Second cron run same month skips already-sent shops
- [ ] Merchant with no data receives valid empty-state email
- [ ] CTA link opens correct shop in main app analytics

---

## Optional Future Enhancement: Screenshot Fallback

If HTML bar charts are not visually acceptable after QA, a secondary approach can capture chart sections from a dedicated read-only report page using Puppeteer/Playwright and embed as `<img>` in the email. This is **not part of v1** and should only be considered if HTML chart visualizations fail design review in Gmail/Outlook. The primary and recommended path remains HTML-only rendering.

---

## References

- Analytics routes: `POST/GET /api/analytics/*` (Express backend)
- Main app analytics proxy: `pages/api/v1/analytics/proxy.js` (unchanged)
- Pro plan filter: `bucks_premium_pro`, `bucks_premium_pro_annual_60`, `bucks_premium_pro_annual_65`
- Non-Pro preview (future): all active merchants not in Pro analytics plans; `oneTimeAnalyticsReportSentAt` for send-once
- Date range: `last_month` (previous calendar month, UTC) for Pro monthly; `last_30_days` TBD for non-Pro preview
- Brevo pattern: `utils/mailEngine.js` (main app — reference for analytics `mailService`)
- Cloud Scheduler auth: `Authorization: Bearer CRON_SECRET`
