# Active Users Retrieval

## Overview

Active users retrieval is a background process that extracts currently active accounts from a datastore, optionally verifies that each account is still accessible through an external API, and writes the result to a downloadable output file such as CSV.

This pattern is useful for operational tasks, migrations, audits, reporting, customer outreach, and any workflow that needs an up-to-date list of valid active accounts without blocking an HTTP request.

## Purpose

Applications often store an `isActive`, `status`, or subscription state locally, but local state can drift from reality. A retrieval process helps produce a reliable active-user list by combining:

- A local database filter for likely active accounts.
- Cursor-based pagination for large datasets.
- Optional external validation using each account's API credentials.
- Batched output writing to avoid high memory usage.
- Background execution so long-running exports do not time out.

This is needed when the account list may contain thousands of records, external validation is slow, or the result needs to be consumed by other operational processes.

## High-Level Flow

1. Start the export from an admin endpoint, CLI command, or scheduled job.
2. Immediately return a response if triggered through HTTP.
3. Query active users in batches using cursor-based pagination.
4. For each batch, process users with a controlled concurrency limit.
5. Skip records missing required identifiers or credentials.
6. Optionally call an external account API to confirm the account is still valid.
7. Append valid records to a CSV or another output sink.
8. Track success and failure counts.
9. Allow the process to be stopped safely between batches or chunks.
10. Expose a download endpoint or storage location for the generated file.

## Key Implementation Details

### Background Execution

For HTTP-triggered exports, return immediately and run the extraction asynchronously. This prevents client, proxy, or platform timeouts.

```ts
app.post("/admin/exports/active-users", async (_req, res) => {
  res.status(202).json({
    status: "processing",
    message: "Active user export started.",
    downloadUrl: "/admin/exports/active-users/download"
  });

  void runActiveUsersExport();
});
```

For production systems, prefer a queue or job runner instead of an in-memory background function when jobs must survive process restarts.

### Cursor-Based Pagination

Use cursor pagination instead of offset pagination for large tables. Cursor pagination avoids performance degradation and duplicate or skipped rows when data changes during the export.

```ts
let lastId: string | undefined;
const batchSize = 1000;

while (true) {
  const users = await db.user.findMany({
    where: { isActive: true },
    select: {
      id: true,
      email: true,
      accountDomain: true,
      accessToken: true,
      planName: true
    },
    orderBy: { id: "asc" },
    take: batchSize,
    ...(lastId ? { cursor: { id: lastId }, skip: 1 } : {})
  });

  if (users.length === 0) break;

  await processUsers(users);
  lastId = users[users.length - 1].id;
}
```

### Controlled Concurrency

External validation can quickly exhaust sockets, hit rate limits, or overload downstream APIs. Process users in small chunks.

```ts
async function processWithConcurrency<T>(items: T[], limit: number, handler: (item: T) => Promise<void>) {
  for (let i = 0; i < items.length; i += limit) {
    const chunk = items.slice(i, i + limit);
    await Promise.all(chunk.map(handler));
  }
}
```

### Optional External Validation

If local active status is not sufficient, call an authoritative external API using the account's credentials. Treat failures as non-fatal and continue processing other users.

```ts
async function verifyAccount(domain: string, accessToken: string): Promise<boolean> {
  const response = await fetch(`https://${domain}/admin/api/account.json`, {
    headers: {
      Authorization: `Bearer ${accessToken}`
    },
    signal: AbortSignal.timeout(5000)
  });

  return response.ok;
}
```

### CSV Output

Write the output incrementally instead of keeping every row in memory. Escape CSV values before writing.

```ts
function csvValue(value: string | null | undefined): string {
  return `"${String(value ?? "").replace(/"/g, '""')}"`;
}

async function appendRows(filePath: string, rows: Array<string[]>) {
  const lines = rows.map((row) => row.map(csvValue).join(","));
  await fs.appendFile(filePath, `${lines.join("\n")}\n`);
}
```

### Stop or Cancel Support

Use a cancellation flag, job status, or queue cancellation token. Check it between batches and before expensive external calls.

```ts
let shouldStop = false;

app.post("/admin/exports/active-users/stop", (_req, res) => {
  shouldStop = true;
  res.json({ message: "Export will stop after the current batch." });
});

while (!shouldStop) {
  const users = await fetchNextBatch();
  if (users.length === 0) break;
  await processUsers(users);
}
```

## Required Configuration

The exact configuration depends on the application, but the pattern typically needs:

| Setting | Purpose |
| --- | --- |
| Database connection | Reads users or accounts. |
| Active-user filter | Defines which records are eligible, such as `isActive = true`. |
| Batch size | Controls database page size. |
| Concurrency limit | Controls simultaneous external checks. |
| Output path or bucket | Stores generated CSV or export file. |
| External API version or base URL | Used only when validating accounts externally. |
| Request timeout | Prevents hanging external validation calls. |

Example environment variables:

```env
DATABASE_URL="postgresql://..."
ACTIVE_USERS_EXPORT_PATH="./exports/active_users.csv"
ACTIVE_USERS_BATCH_SIZE="1000"
ACTIVE_USERS_CONCURRENCY="10"
EXTERNAL_API_VERSION="2026-01"
```

## Important Considerations and Edge Cases

- **Long-running jobs:** Use a queue or worker for exports that must survive restarts.
- **Duplicate starts:** Prevent multiple exports from writing to the same file at the same time.
- **Large datasets:** Use cursor pagination and incremental file writes.
- **Missing credentials:** Skip records that cannot be externally verified and count them as failures or skipped rows.
- **External API failures:** Add timeouts, retries where appropriate, and continue processing other accounts.
- **Rate limits:** Keep concurrency low and add pauses between large batches.
- **CSV correctness:** Escape quotes, commas, and newlines. Always write a header row.
- **Sensitive data:** Avoid exporting access tokens or secrets. Limit CSV columns to operationally necessary fields.
- **Partial results:** A stopped or failed export may still produce a useful partial file. Mark the job status clearly.
- **Process-local state:** In-memory stop flags and progress counters do not work across multiple server instances.

## Integrating in Another Application

1. Identify the account table and active-status field.
2. Decide whether local state is enough or if external validation is required.
3. Create a background job or admin endpoint to start the export.
4. Implement cursor-based database pagination.
5. Add controlled concurrency for per-account work.
6. Write output incrementally to a file, object storage, or database table.
7. Track progress, successes, failures, and stop state.
8. Expose a secure download endpoint or storage link.
9. Protect all export endpoints with admin authentication.

## Minimal Reusable Example

```ts
type ActiveUser = {
  id: string;
  email: string | null;
  accountDomain: string | null;
  accessToken: string | null;
  planName: string | null;
};

async function runActiveUsersExport() {
  const outputPath = "./exports/active_users.csv";
  const batchSize = 1000;
  const concurrency = 10;
  let lastId: string | undefined;

  await fs.writeFile(outputPath, "Email,Account Domain,Plan\n");

  while (true) {
    const users: ActiveUser[] = await db.user.findMany({
      where: { isActive: true },
      orderBy: { id: "asc" },
      take: batchSize,
      ...(lastId ? { cursor: { id: lastId }, skip: 1 } : {})
    });

    if (users.length === 0) break;

    const rows: string[][] = [];

    await processWithConcurrency(users, concurrency, async (user) => {
      if (!user.accountDomain || !user.accessToken) return;

      const isValid = await verifyAccount(user.accountDomain, user.accessToken);
      if (!isValid) return;

      rows.push([user.email ?? "", user.accountDomain, user.planName ?? ""]);
    });

    await appendRows(outputPath, rows);
    lastId = users[users.length - 1].id;
  }
}
```

## Recommended API Endpoints

```http
POST /admin/exports/active-users
POST /admin/exports/active-users/stop
GET  /admin/exports/active-users/progress
GET  /admin/exports/active-users/download
```

Use `POST` for actions that start or stop work. Use `GET` only for read-only status and download endpoints.
