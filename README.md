# Legacy ScriptTag cleanup

For developers familiar with Postman and Shopify Admin APIs, implementing or running a legacy ScriptTag cleanup.

Removes the legacy ScriptTags in a configured target list after their replacement app embeds are active on the published themes. Each target contains the shop domain, ScriptTag ID, and expected script URL. The API does not verify app embed activation.

The workflow applies to other apps; the paths, token storage, and configuration below describe this implementation and must be adapted when porting it. Set `N` to the number of entries in your target list when checking results.

## Implementation

- Reads `users.accessToken` from MongoDB by `myshopify_domain`. **No database writes.**
- Uses Shopify Admin GraphQL: `scriptTag` reads the exact ID and URL; `scriptTagDelete` removes it. Numeric IDs become `gid://shopify/ScriptTag/<id>`.
- Uses `SHOPIFY_API_VERSION`, defaulting to `2026-01`.
- One POST processes the full target list in batches of 5, with a 1-second gap between batches. Progress appears in server logs.
- Every POST starts fresh and generates a report ID. Results are saved locally as JSON; CSV is generated on download.

## Batching and delay

One POST processes the entire target list; callers do not send a request per shop.
The same scheduling applies to dry run, deletion, and cross-verification:

1. Take up to **5 shops** and process them concurrently using `Promise.allSettled`.
2. Wait for every shop in that batch to finish, then save its results in target-list order.
3. Wait **1 second (`1000` ms)** before starting the next batch. There is no delay after the final batch.

Each shop's lookup, optional deletion, and verification run sequentially within
its worker. The final batch may contain fewer than 5 shops. Logs show each shop's
start and result; the POST returns after the full list finishes.

Batching bounds concurrency and spaces requests, but does not guarantee Shopify
will never throttle them. Total duration includes API calls, report writes, and
`ceil(N / 5) - 1` inter-batch pauses. This is synchronous processing, not a background job.

To tune this implementation, change `targets.slice(start, start + 5)` and
`sleep(1000)` in `utils/admin/legacyScriptTags/api.js`, and update the batch tests.

## Shopify GraphQL operations

The backend sends both operations to:

```text
POST https://<shop>.myshopify.com/admin/api/<SHOPIFY_API_VERSION>/graphql.json
X-Shopify-Access-Token: <that shop's stored access token>
Content-Type: application/json
```

The cleanup Bearer Token authenticates Postman to your app; the merchant access
token authenticates your backend to Shopify. Do not expose merchant tokens in reports.
Lookup requires ScriptTag read access; deletion requires `write_script_tags`.

**Lookup: used by dry run, before deletion, and after deletion.**

```graphql
query CleanupScriptTag($id: ID!) {
  scriptTag(id: $id) {
    id
    src
  }
}
```

Convert the numeric ID from the target list to a Shopify global ID. Both operations
use the same variables format (example ID):

```json
{
  "id": "gid://shopify/ScriptTag/115729858596"
}
```

If a tag is returned, compare both `id` and `src` against the configured target.
A mismatch prevents deletion.

**Deletion: sent only in `apply` mode after a matching lookup.**

```graphql
mutation CleanupDeleteScriptTag($id: ID!) {
  scriptTagDelete(id: $id) {
    deletedScriptTagId
    userErrors {
      field
      message
    }
  }
}
```

Require empty `userErrors` and the expected `deletedScriptTagId`, then repeat the
lookup. A successful response confirming absence is:

```json
{
  "data": {
    "scriptTag": null
  }
}
```

This produces `deleted` during apply verification or `already_absent` during a
fresh dry run. HTTP failures and top-level GraphQL `errors` are errors, not proof
of absence, even if the response also contains null data.

Reference: [scriptTagDelete](https://shopify.dev/docs/api/admin-graphql/2026-01/mutations/scriptTagDelete).

## Setup in Postman

Set `baseUrl` to the deployed app's HTTPS origin. The server must have `MONGO_URI` for the merchant database and `LEGACY_SCRIPT_TAG_CLEANUP_SECRET` (at least 32 characters).

Use these headers for POST requests; downloads need only Authorization:

```http
Authorization: Bearer <value of LEGACY_SCRIPT_TAG_CLEANUP_SECRET>
Content-Type: application/json
```

## 1. Dry run: check before deletion

**POST** `{{baseUrl}}/api/internal/legacy-script-tags/run`

Body (raw JSON):

```json
{ "mode": "dry_run" }
```

Queries Shopify for each listed tag. Exact ID/URL matches produce `would_delete`; no mutation is sent.

Expected when all listed tags are present and match: `processed` equals `N`, `done` is `true`, and `counts.would_delete` equals `N`. Review any other statuses before applying.

## 2. Apply: delete matching tags

Use the **same POST endpoint and headers** with:

```json
{ "mode": "apply" }
```

Rechecks each ID and URL, calls `scriptTagDelete`, then queries the ID again. A successful lookup returning `null` confirms absence and produces `deleted`.

Expected when all listed tags were present: `counts.deleted` equals `N`. Already-removed tags are skipped. This affects the targeted merchant stores even when called through staging. The API has no undo operation.

## 3. Cross-verify: dry run again

Use the **same POST endpoint** with:

```json
{ "mode": "dry_run" }
```

This makes fresh Shopify queries, not reads of the previous report. Expected:

- `processed` and `total` equal `N`.
- `done` is `true`.
- `counts.already_absent` equals `N`.

These fields appear under `data` in the response. API failures never count as absence. Also check currency selection and price conversion on affected storefronts: tag absence alone does not prove the app embed works.

## Download each report

Each POST response includes `data.downloads.json` and `data.downloads.csv`. Prefix the returned path with `baseUrl`, then send **GET** using the same Bearer Token. No body is needed.

```text
GET {{baseUrl}}/api/internal/legacy-script-tags/report?runId=<ID returned by POST>&format=json
```

Use `format=csv` for CSV. Downloads read local files only, without calling Shopify or MongoDB. Save the dry-run, apply, and cross-verification reports before redeploying.

## Important limits and errors

- Reports live in `<OS temporary directory>/bucks-legacy-script-tag-reports` and can disappear on container replacement. Use one app instance: files and the execution lock are not shared across replicas.
- Requests are synchronous. On timeout, check logs and the partial report link before retrying; there is no automatic resume. A retry starts a fresh check of all targets.
- `done: true` means processing finished, not that every shop succeeded. Investigate `mismatch`, `missing_session`, `error`, or `unverified`. `absent_after_error` means a mutation response failed but a subsequent lookup confirmed absence.
- HTTP 401: incorrect token; 503: cleanup secret not configured; 409: another cleanup is running on this process; 404 on download: report not found on this instance.

## Code and tests

For another app, replace the target list with that app's shop/ID/URL entries and
use its own stored merchant access tokens. Adapt the token lookup, route paths,
authentication configuration, and tests to that app. Confirm its app embed works
independently before deleting the legacy tags.

- Target list: `utils/admin/legacyScriptTags/targets.json`.
- Routes: `pages/api/internal/legacy-script-tags/run.js` and `report.js`.
- Authentication, batches, local reports: `utils/admin/legacyScriptTags/api.js`.
- Shopify lookup, deletion, verification: `utils/admin/legacyScriptTags/processor.js`.

```sh
yarn test tests/admin/legacyScriptTagsApi.test.js tests/admin/legacyScriptTagsProcessor.test.js
```

Tests mock Shopify/database access; they do not perform live deletion.
