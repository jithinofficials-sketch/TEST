# Webhook Update System

## Overview

A webhook update system bulk-updates webhook subscription callback URLs across many tenant accounts. It reads target accounts from a source such as CSV or a database, looks up each account's API credentials, finds existing webhook subscriptions by topic, updates callback URLs when needed, and records the result for audit and retry purposes.

This pattern is useful when an application changes webhook infrastructure, migrates domains, standardizes callback routes, or needs to repair webhook subscriptions across installed accounts.

## Purpose

Webhook URLs often need to change when infrastructure changes. Updating them manually account-by-account is slow and error-prone. A bulk update system provides a safe, repeatable workflow that:

- Processes many accounts without blocking a request.
- Updates only existing webhook subscriptions.
- Skips subscriptions that already point to the correct URL.
- Controls API concurrency and rate-limit pressure.
- Tracks progress and per-topic outcomes.
- Supports stopping the migration safely.
- Produces an audit file for completed, skipped, and failed updates.

## High-Level Flow

1. Define a mapping of webhook topics to their new callback URLs.
2. Start a background update job from an admin endpoint, CLI command, or scheduled worker.
3. Read account identifiers from CSV, database query, or queue input.
4. Load active accounts and API credentials from the datastore.
5. Optionally filter accounts by install date, plan, region, or migration eligibility.
6. Process accounts in batches with a fixed concurrency limit.
7. For each account and topic, query the external platform for existing webhook subscriptions.
8. Skip missing subscriptions or subscriptions already using the target URL.
9. Update outdated subscriptions through the platform API.
10. Record `updated`, `skipped`, or `failed` for each account-topic pair.
11. Expose progress, stop, and audit-download endpoints.

## Key Implementation Details

### Topic-to-URL Mapping

Keep webhook topics and callback URLs in configuration, not embedded directly in business logic.

```ts
const WEBHOOK_TARGETS: Record<string, string> = {
  ACCOUNT_UPDATED: "https://hooks.example.com/webhooks/account-updated",
  SUBSCRIPTION_UPDATED: "https://hooks.example.com/webhooks/subscription-updated",
  APP_UNINSTALLED: "https://hooks.example.com/webhooks/app-uninstalled"
};
```

For multi-environment deployments, load these values from environment variables or a configuration service.

### Background Runner

Return immediately when starting from HTTP, then run the update asynchronously.

```ts
app.post("/admin/webhooks/update", async (req, res) => {
  const limit = Number(req.query.limit ?? 100);

  if (webhookUpdateState.isRunning) {
    return res.status(409).json({ error: "Webhook update already running." });
  }

  webhookUpdateState.start(limit);
  void runWebhookUpdate({ limit });

  res.status(202).json({
    status: "running",
    progressUrl: "/admin/webhooks/update/progress",
    stopUrl: "/admin/webhooks/update/stop"
  });
});
```

For production-critical migrations, prefer a durable job queue over in-memory state.

### Input Source

The update job can read accounts from CSV, a database, or a queue. CSV is convenient when the target set comes from a previous export.

```ts
async function* readAccountDomains(filePath: string): AsyncGenerator<string> {
  const stream = fs.createReadStream(filePath);
  const lines = readline.createInterface({ input: stream, crlfDelay: Infinity });

  for await (const line of lines) {
    const value = line.trim();
    if (!value || value.toLowerCase() === "account domain") continue;
    yield value;
  }
}
```

### Credential Lookup and Eligibility Filtering

Use the input source only as a list of account identifiers. Fetch credentials from the database at processing time and filter to eligible accounts.

```ts
async function loadEligibleAccounts(domains: string[], cutoffDate: Date) {
  return db.account.findMany({
    where: {
      domain: { in: domains },
      isActive: true,
      installedAt: { lt: cutoffDate }
    },
    select: {
      domain: true,
      accessToken: true
    }
  });
}
```

Eligibility filters are optional, but they are useful when only older installations or specific cohorts need migration.

### Webhook Lookup and Update

The exact API depends on the platform. The reusable pattern is:

1. Find webhook subscriptions by topic.
2. Read the current callback URL.
3. Skip if the current URL already matches the target URL.
4. Update the subscription by ID.
5. Capture API validation errors.

Example GraphQL-style lookup:

```ts
const findWebhookQuery = `
  query FindWebhooks($topic: WebhookTopic!) {
    webhookSubscriptions(first: 10, topics: [$topic]) {
      edges {
        node {
          id
          topic
          endpoint {
            __typename
            ... on WebhookHttpEndpoint {
              callbackUrl
            }
          }
        }
      }
    }
  }
`;
```

Example GraphQL-style update:

```ts
const updateWebhookMutation = `
  mutation UpdateWebhook($id: ID!, $webhookSubscription: WebhookSubscriptionInput!) {
    webhookSubscriptionUpdate(id: $id, webhookSubscription: $webhookSubscription) {
      userErrors {
        field
        message
      }
    }
  }
`;
```

Reusable update function:

```ts
async function updateWebhookTopic(client: ApiClient, topic: string, targetUrl: string) {
  const result = await client.request(findWebhookQuery, { topic });
  const subscriptions = result.webhookSubscriptions?.edges ?? [];

  if (subscriptions.length === 0) {
    return { status: "skipped", reason: "subscription_not_found" };
  }

  const subscription = subscriptions[0].node;
  const currentUrl = subscription.endpoint?.callbackUrl;

  if (currentUrl === targetUrl) {
    return { status: "skipped", reason: "already_current" };
  }

  const updateResult = await client.request(updateWebhookMutation, {
    id: subscription.id,
    webhookSubscription: { callbackUrl: targetUrl }
  });

  const userErrors = updateResult.webhookSubscriptionUpdate?.userErrors ?? [];
  if (userErrors.length > 0) {
    return { status: "failed", reason: JSON.stringify(userErrors) };
  }

  return { status: "updated" };
}
```

### HTTP Client Reliability

For large migrations, configure the HTTP client intentionally:

- Reuse connections with keep-alive where supported.
- Set request timeouts.
- Retry transient network errors.
- Retry rate-limit responses with backoff.
- Limit concurrent sockets or concurrent requests.

```ts
const agent = new https.Agent({
  keepAlive: true,
  maxSockets: 50
});
```

### Progress Tracking

Track progress in a shared state object, database row, cache key, or job record.

```ts
type WebhookUpdateProgress = {
  status: "idle" | "running" | "stopped" | "finished" | "error";
  totalRows: number;
  processedRows: number;
  successfulAccounts: number;
  failedAccounts: number;
  errors: string[];
  startedAt: Date | null;
  endedAt: Date | null;
};
```

Example progress response:

```json
{
  "status": "running",
  "progress": "42%",
  "stats": {
    "totalRows": 1000,
    "processedRows": 420,
    "successfulAccounts": 400,
    "failedAccounts": 20
  },
  "errors": []
}
```

### Audit Output

Write one row per account-topic attempt. This makes retries and investigation easier.

```csv
account,topic,status,timestamp,reason
example.myplatform.com,ACCOUNT_UPDATED,updated,2026-07-11T10:00:00.000Z,
example.myplatform.com,APP_UNINSTALLED,skipped,2026-07-11T10:00:01.000Z,already_current
example.myplatform.com,SUBSCRIPTION_UPDATED,failed,2026-07-11T10:00:02.000Z,rate_limited
```

## Required Configuration

| Setting | Purpose |
| --- | --- |
| API version or base URL | Selects the external platform API endpoint. |
| Webhook topic map | Defines each topic and desired callback URL. |
| Input source path or query | Identifies accounts to process. |
| Database connection | Loads account credentials and eligibility fields. |
| Batch size | Controls how many input accounts are processed per batch. |
| Concurrency limit | Controls simultaneous account updates. |
| Request timeout | Prevents individual API calls from hanging. |
| Retry count and backoff | Handles rate limits and transient network failures. |
| Audit output path | Stores migration results. |

Example environment variables:

```env
WEBHOOK_UPDATE_BATCH_SIZE="25"
WEBHOOK_UPDATE_LIMIT="1000"
WEBHOOK_UPDATE_INPUT_PATH="./exports/active_accounts.csv"
WEBHOOK_UPDATE_AUDIT_PATH="./exports/webhook_update_audit.csv"
EXTERNAL_API_VERSION="2026-01"
WEBHOOK_ACCOUNT_UPDATED_URL="https://hooks.example.com/webhooks/account-updated"
WEBHOOK_SUBSCRIPTION_UPDATED_URL="https://hooks.example.com/webhooks/subscription-updated"
WEBHOOK_APP_UNINSTALLED_URL="https://hooks.example.com/webhooks/app-uninstalled"
```

## Important Considerations and Edge Cases

- **Idempotency:** Always skip webhook subscriptions that already use the desired callback URL.
- **Missing subscriptions:** Decide whether missing webhooks should be skipped, created, or reported as failures. The safest migration behavior is usually to skip unless creation is explicitly required.
- **Multiple subscriptions per topic:** Define whether to update the first match, all matches, or only HTTP endpoint subscriptions.
- **Non-HTTP endpoints:** Some platforms support event bus or pub/sub endpoints. Verify the endpoint type before reading or changing callback URLs.
- **Rate limits:** Use low concurrency, retry `429` responses with backoff, and add pauses between batches.
- **Network instability:** Use timeouts, retries, and keep-alive connections for large runs.
- **Credential drift:** Access tokens may be expired or revoked. Count these as failures and continue.
- **Partial migrations:** Persist audit results so the job can be resumed or retried safely.
- **Process restarts:** In-memory progress is lost on restart. Store progress externally for critical migrations.
- **Security:** Protect start, stop, progress, and download endpoints with admin-only authorization.
- **Dry runs:** Consider a dry-run mode that reports intended changes without updating subscriptions.

## Integrating in Another Application

1. Define the webhook topics and target callback URLs.
2. Choose the input source: database query, CSV export, queue, or static list.
3. Implement account credential lookup from your datastore.
4. Add eligibility filters if only a subset should be migrated.
5. Implement platform-specific webhook lookup and update calls.
6. Add batching, concurrency limits, timeouts, and retries.
7. Track progress and expose it through an admin-safe interface.
8. Write audit rows for every account-topic operation.
9. Add stop/cancel support that exits safely between batches.
10. Run a limited pilot before increasing the processing limit.

## Minimal Reusable Example

```ts
async function runWebhookUpdate(options: { limit: number; cutoffDate: Date }) {
  const batchSize = 25;
  let batch: string[] = [];
  let readCount = 0;

  for await (const domain of readAccountDomains("./exports/active_accounts.csv")) {
    if (readCount >= options.limit || webhookUpdateState.shouldStop) break;

    batch.push(domain);
    readCount++;

    if (batch.length >= batchSize) {
      await processWebhookBatch(batch, options.cutoffDate);
      batch = [];
      await delay(2000);
    }
  }

  if (batch.length > 0 && !webhookUpdateState.shouldStop) {
    await processWebhookBatch(batch, options.cutoffDate);
  }
}

async function processWebhookBatch(domains: string[], cutoffDate: Date) {
  const accounts = await loadEligibleAccounts(domains, cutoffDate);

  await Promise.all(accounts.map(async (account) => {
    const client = createApiClient(account.domain, account.accessToken);

    for (const [topic, targetUrl] of Object.entries(WEBHOOK_TARGETS)) {
      const result = await updateWebhookTopic(client, topic, targetUrl);
      await appendAuditRow({
        account: account.domain,
        topic,
        status: result.status,
        reason: result.reason ?? "",
        timestamp: new Date().toISOString()
      });
    }
  }));
}
```

## Recommended API Endpoints

```http
POST /admin/webhooks/update
POST /admin/webhooks/update/stop
GET  /admin/webhooks/update/progress
GET  /admin/webhooks/update/audit.csv
```

Use `POST` for operations that start or stop a migration. Use `GET` for read-only progress and audit downloads.

## Rollout Strategy

1. Run a dry run for a small sample and inspect the planned changes.
2. Run the update with a low limit, such as 10 or 25 accounts.
3. Check audit output and platform logs.
4. Increase the limit gradually.
5. Re-run failed accounts after fixing credential, rate-limit, or platform issues.
6. Keep the audit file for rollback analysis and compliance records.
