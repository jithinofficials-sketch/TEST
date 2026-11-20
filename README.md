# Legacy ScriptTag Cleanup

## Approved approach

Run a one-time Node script on the production server to remove only the legacy
BUCKS ScriptTag registrations identified in the merchant-provided sheet.
Use fixed JSON input with shop domain, ScriptTag ID as a string, and expected
script URL. The pasted list contains 46 entries. Do not include merchant names,
email addresses, or other unnecessary personal data.

The merchant confirms app embeds are enabled for these shops. This cleanup
uses that confirmation; it does not independently verify app embed state.
No stores, sessions, settings, theme assets, or app embed blocks are deleted.

## Files and responsibilities

- `scripts/data/legacy-script-tags-2026-11-20.json`: the 46 exact shop/ID/URL triples.
- `scripts/cleanup-legacy-script-tags.js`: executable CLI, input validation,
  per-shop processing, Shopify requests, and report output; export injectable
  processing functions for isolated tests without loading Prisma on import.
- `tests/scripts/cleanupLegacyScriptTags.test.js`: mocked database and HTTP tests.
- `docs/operations/legacy-script-tag-cleanup.md`: production execution and report
  interpretation, including container file-copy requirements.

## Execution contract

Default execution is a dry run. Only `--apply` permits mutations. Reject unknown
arguments. `--report <path>` selects a new JSONL output file; create it exclusively
before database or Shopify operations so existing reports cannot be overwritten.
Provide `--help` without connecting to the database.

Use Node 22.23.5 and the existing generated Prisma client. Load production
configuration through Node's `--env-file` option or inherited environment.
Require `MONGO_URI` and use the existing `utils/prisma.js` singleton via a dynamic
import when executing the CLI. Disconnect in a finally block. Do not write to
the database or print tokens, connection strings, or raw upstream error bodies.

Validate the complete fixed input before processing: nonempty array, canonical
myshopify.com domains, decimal string IDs, HTTPS script URLs with the exact
`/apps/buckscc/sdk.min.js` path, and unique shop/ID pairs. Reject invalid input
before any remote operation.

## Per-shop flow

1. Read the session by its unique shop domain, selecting only the token and
   session metadata needed for processing. Missing tokens are reported separately.
2. Fetch the exact ScriptTag using its Shopify global ID. A successful lookup
   returning null means already absent; API errors never mean absent.
3. Compare both returned ID and script URL with the fixed input. A mismatch
   prevents deletion and is included in the report.
4. In dry-run mode, report `would_delete` without issuing a mutation.
5. In apply mode, call Admin GraphQL `scriptTagDelete`. Inspect HTTP failures,
   top-level GraphQL errors, mutation user errors, and the returned deletion ID.
6. Read the same ID again to confirm absence before reporting `deleted`.
   Failed verification is reported as unverified, not as a confirmed deletion.

Use Admin API version `2026-10`, which supports the validated deletion mutation.
Read access and `write_script_tags` are required. Actual API responses determine
access; stored scope metadata alone is not authoritative.

Process sequentially. Use request timeouts and bounded retries for throttled
requests, honoring Retry-After where present. Avoid blindly retrying ambiguous
mutation failures: read the tag to reconcile whether it still exists. Never
interpret transport failures as successful deletion. Individual shop failures
do not prevent the remaining shops from being processed.

## Reporting and reruns

Write each result to JSONL immediately, including timestamp, mode, shop, ID,
expected URL, status, and sanitized error category. Print a final count by status
and report location. Exit nonzero for unresolved failures or mismatches.
If report persistence fails, stop processing further shops.

Reruns are supported: successfully removed tags are reported as already absent.
Reports contain no credentials or owner contact details.

## Production packaging

The current production Docker image omits `scripts/` and `utils/`. Document
copying the cleanup script, fixed input, and `utils/prisma.js` into their matching
paths under `/app` in the existing container, where package.json, dependencies,
generated Prisma client, and `.env` already exist. Alternatively, run from a
server checkout with those prerequisites. No Dockerfile change is required for
this one-time operation. Record the exact report path so container-local results
can be copied out before the container is replaced.

## Verification

Mock all remote operations in automated tests. Cover dry run issuing zero
mutations, matching deletion plus verification, URL/ID mismatch, already absent,
missing session, HTTP and GraphQL failures, user errors, throttling, ambiguous
mutation reconciliation, failed verification, continuation after shop failure,
input validation, and refusal to overwrite a report. Assert the fixed input has
46 unique entries and compare its triples with the original pasted list.

Validate generated GraphQL operations against Shopify's schema. Run focused
tests and CLI help locally without production credentials. Production execution
and storefront checks are separate operational steps, not implied by unit tests.

## Source references

- `utils/sessionHandler.js`: stored Shopify sessions and access tokens.
- `utils/prisma.js`: application Prisma singleton.
- `prisma/schema.prisma`: unique session shop and MONGO_URI configuration.
- `package.json`: ESM modules, Node version, and existing test dependencies.
- `Dockerfile`: production container contents.
- https://shopify.dev/docs/api/admin-graphql/2026-10/mutations/scriptTagDelete
