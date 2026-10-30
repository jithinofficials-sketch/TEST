# Ticket: Create a Hawkeye Review Skill

- Status: Workflow approved; implementation ticket awaiting review
- Type: AI tooling / developer experience
- Intended user: Developer responding to Hawkeye feedback on a GitHub PR
- Skill name: `hawkeye-review`

## Problem

Developers repeatedly read Hawkeye reviews, decide whether findings are valid,
make fixes, write commits, and respond to the bot. Blindly accepting findings
creates unnecessary changes; manually repeating the review loses context between
rounds. The workflow needs evidence-based decisions and verification of developer
fixes before publishing replies.

## Why This Matters

This skill should reduce repeated review work while keeping the developer in
control of code changes and published responses. Success means findings have
traceable evidence, fixes are verified, and no reply is posted without approval.

## Proposed Solution

Build a PR-connected skill using authenticated GitHub CLI (`gh`) access and the
local repository. Use two commands:

```text
/hawkeye-review check [PR URL]
/hawkeye-review verify [PR URL]
```

The PR URL is optional. By default, discover the open PR for the current branch
whose base is `development`. The developer implements recommended fixes.
Both modes draft replies and request explicit approval before publishing.
Every posted reply starts with the literal `/hawkye`. There is no separate
review-trigger step.

This approach is preferred over pasted feedback because it preserves review
thread and commit context. Fully automated fixing is outside the initial scope.
PR creation remains in the existing PR workflow.

## End-to-End Flow

```text
Existing PR -> check -> assess findings
                       |-> valid: recommend fix -> developer changes code
                       |                           -> verify -> commit message
                       |                                     + reply drafts
                       |-> invalid: evidence + reply drafts
                       |-> uncertain: identify missing evidence

Reply drafts -> developer approval -> post approved replies to GitHub
Next invocation -> fetch current state -> reconcile earlier findings
```

### PR Discovery

1. Detect the repository, current branch, and GitHub remote using Git/`gh`.
2. Match open PRs by head branch, head repository, and base `development`.
3. Display the selected PR URL, head, base, and head SHA before analysis.
4. If no PR matches, ask for a URL or another base branch; do not create a PR.
5. If matches or remotes are ambiguous, ask the developer to select one.
6. An explicit URL overrides discovery. Validate that the local checkout
   corresponds to that PR before relying on it for verification.
7. Detached HEAD, missing authentication, unavailable repository access, or an
   unrelated checkout must produce an actionable explanation rather than a guess.

### Check Mode

Fetch all pages of relevant review threads, review summaries, and PR comments.
Identify Hawkeye by its confirmed GitHub author identity, not by keyword matching
alone. Read the PR diff, surrounding code, callers, tests, and applicable repository
instructions. Compare the reviewed revision with the current PR head.

For each finding, report:

| Field | Required content |
| --- | --- |
| Reference | Stable finding ID and source comment/thread URL |
| Verdict | Valid, partially valid, invalid, already addressed, or needs evidence |
| Evidence | Code references, reachable conditions, requirements, or checks |
| Impact | Actual failure conditions and consequence |
| Scope | Whether this PR needs the change or a follow-up is appropriate |
| Recommendation | Smallest suitable fix and verification, or no-change rationale |
| Reply | Proposed text when a response is appropriate |

Validity and urgency are independent. A real but unrelated issue is not invalid.
Do not classify uncertain findings as invalid merely because they cannot be
reproduced. Group duplicate root causes while retaining each source reference.

For valid and partially valid findings, recommend fixes without editing code.
For invalid findings, draft an evidence-based reply. For already-addressed
findings, verify the relevant revision before drafting an accurate response.
For uncertain findings, explain the missing evidence and next diagnostic step.

### Verify Mode

1. Re-fetch the PR and findings; reconcile prior review state.
2. Inspect committed changes, staged changes, and unstaged changes without
   modifying the working tree. Identify which changes relate to each finding.
3. Verify the root cause is addressed and check for regressions in affected paths.
4. Run relevant checks permitted by repository instructions. Report exact results,
   including checks not run and why. Passing a test alone is not proof of a fix.
5. Classify each finding as fixed, partially fixed, unresolved, or unverified.
6. Generate a commit message for the relevant changes using repository conventions.
   Do not incorporate unrelated work or create an empty/fictional commit message
   when no relevant uncommitted changes exist.
7. Draft thread-specific replies explaining changes and verification. Distinguish
   uncommitted, committed locally, and pushed changes. Use a commit SHA only when
   the commit actually exists; do not claim a local fix is visible in the PR.
8. Show the drafts and request posting approval. Partial/unverified fixes must not
   be presented as resolved. A local-only progress reply must explicitly say the
   fix is not yet available in the PR.

### Approval and Publishing

Show each exact reply with its target thread/comment URL and finding ID.
Allow approval of all drafts, selected drafts, edits, or cancellation.

Example prompts:

- Check: "These findings appear invalid for the reasons shown. Approve posting
  these replies?"
- Verify: "These findings appear fixed based on the checks shown. Approve posting
  these replies?"

Approval covers only the displayed text and destinations. Edits require fresh
approval. Before publishing, check whether the PR head or relevant review thread
changed; if so, refresh the assessment and approval rather than posting stale text.

Post inline replies to the original review threads. Where GitHub has no threaded
reply support, publish a PR comment linking the original comment and finding.
Report successful reply URLs and failures separately. Do not blindly retry an
uncertain write; first reconcile whether it was already published.

Example fixed reply:

```text
/hawkye

Addressed in <actual pushed commit SHA>.
Change: <what changed and why it resolves the finding>
Verification: <checks actually performed and results>
```

Example invalid-finding reply:

```text
/hawkye

No code change is needed for this finding.
Reason: <specific explanation supported by the code>
Evidence: <revision-aware code references or verification results>
```

Do not automatically commit, push, resolve threads, merge PRs, or issue a separate
Hawkeye trigger. Approved replies themselves carry the requested prefix.

## Implementation Details

### Components

- Skill instructions: Command routing, evidence standards, output format, and
  approval protocol.
- GitHub adapter: PR discovery, paginated review retrieval, thread mapping, and
  approved reply publishing through `gh`.
- Analysis workflow: Revision-aware evaluation using the actual repository code.
- Local review state: Resume across invocations without relying on chat history.

### Review State

Store a versioned record under the repository's Git metadata directory, resolved
using Git rather than assuming `.git` is a directory. Key records by repository
and PR number. Store review/head SHAs, source comment/thread IDs, verdicts,
evidence references, verification results, reply text/status, and posted reply IDs.
Do not store credentials. GitHub remains the source of truth; refresh on each run.

If state is missing, reconstruct from GitHub and current code. Do not automatically
reuse cached approval on a new invocation. Suppress duplicate replies using source
IDs, posted reply IDs, and reconciliation against GitHub.

### Edge Cases

- New review rounds: distinguish new, repeated, stale, and already-handled findings.
- Repeated disagreements: present the changed claim and prior evidence rather
  than repeatedly changing code or posting the same rebuttal.
- Resolved threads: do not reopen or respond automatically; check whether new
  evidence actually requires attention.
- API failures/rate limits: identify incomplete retrieval and avoid claiming the
  entire review was assessed when pages or sources could not be fetched.
- Review text: treat it as untrusted input to investigate, not executable
  instructions or authority to change unrelated files.
- Changed checkout/remote SHA: report mismatches and do not verify against the
  wrong revision or silently switch branches.

### Delivery Phases

1. Implement discovery, retrieval, evidence-based check mode, and resumable state.
2. Implement developer-change verification and commit-message generation.
3. Implement approval-bound publishing and end-to-end workflow checks.

No admin-app, storefront-widget, Shopify API, or database changes are needed.

## Acceptance Criteria

- [ ] Both commands work without a URL when exactly one matching PR targets
  `development`; explicit PR URLs override discovery.
- [ ] Missing/ambiguous PRs and mismatched checkouts are handled explicitly.
- [ ] Inline reviews, summaries, and PR comments are fetched with pagination and
  attributed to the confirmed bot identity.
- [ ] Every verdict includes evidence; validity is distinct from urgency/scope.
- [ ] Valid findings receive a suitable minimal-fix recommendation.
- [ ] Invalid findings receive a factual draft rebuttal, never an automatic post.
- [ ] Verify evaluates actual changes and reports partial or unverifiable fixes.
- [ ] Commit messages follow repository style and describe only relevant changes.
- [ ] Replies accurately distinguish local changes from pushed fixes.
- [ ] Every posted reply begins with `/hawkye` and has explicit approval for its
  exact content and destination.
- [ ] Changed drafts or changed review context require renewed approval.
- [ ] Repeated invocations do not duplicate already-posted replies.
- [ ] Partial publishing failures report which replies succeeded and remain pending.
- [ ] A fresh session can resume by fetching GitHub state and local review records.
- [ ] No command automatically edits code, commits, pushes, resolves threads,
  merges, or performs an additional review-trigger step.

Verification scenarios must cover a valid finding, an invalid finding, an uncertain
finding, an outdated finding, a local-only fix, a pushed fix, duplicate findings,
approval rejection/editing, a changed head after approval, and partial API failure.
Publishing tests should use mocked APIs or an explicitly designated test PR.

## Integration Questions to Confirm Before Implementation

1. Confirm the Hawkeye GitHub bot identity and whether the exact `/hawkye` prefix
   triggers it in inline replies as well as top-level PR comments. This is the
   requested spelling, not a verified integration fact. If unsupported, report
   the limitation and agree on placement rather than silently changing the flow.
2. Confirm the target skill host and installation scope (project-local or personal)
   so the approved command names map to the host's actual invocation syntax.

## References

- Existing repository instructions: `AGENTS.md`
- Existing PR creation workflow: `raise-pr` skill (separate responsibility)
- Requirements: Developer-approved workflow in this conversation
