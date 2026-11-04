# What GitHub's APIs offer sous-chef (issues, PRs, reviews, CI)

Research for [#5](https://github.com/wkimdkim-nz/sous-chef/issues/5), part of the map in #1.
Researched 2026-11-04 against docs.github.com (REST and GraphQL references), the `gh` CLI manual
and the Octokit repositories. Every claim cites the page it came from; "Inference" marks
conclusions that no page states.

GitHub plays two roles for sous-chef: one possible **Tracker** (GitHub Issues, maybe with a
Projects v2 board), and the only code host (PRs, reviews, CI).

## TL;DR

1. **Polling is fine and cheap with conditional requests.** A 304 response to a request that carries
   `If-None-Match`/`If-Modified-Since` and an `Authorization` header does not count against the
   primary rate limit. The budget is 5,000 REST requests/hour and 5,000 GraphQL points/hour per user.
   Search is capped separately at 30 requests/minute. [B1][R1][G1][S1]
2. **The Ready signal.** A label (`label:ready`) works with plain REST list/search filters and any
   token type. A Projects v2 `Status` field works through GraphQL (`items(query: "status:Ready")`)
   or the newer Projects REST API, but **fine-grained PATs cannot access Projects owned by a user
   account**. `wkimdkim-nz` is a user account, so a Projects-based Ready signal there needs a
   classic PAT or OAuth token with the `project` / `read:project` scope. [P1][P2][P3][T1]
3. **Review threads and their resolved state exist only in GraphQL.** `PullRequestReviewThread.isResolved`,
   `isOutdated`, `resolveReviewThread`, and `addPullRequestReviewThreadReply` have no REST equivalent.
   REST review comments carry no resolved state, and `gh pr view --json` does not expose `reviewThreads`.
   [Q1][RC1][GH3]
4. **"Approved + green" can be one GraphQL query.** It's
   `PullRequest.reviewDecision == APPROVED` plus `commits(last:1).statusCheckRollup.state == SUCCESS`
   (the rollup covers both check runs and commit statuses), with `mergeStateStatus` (`CLEAN`,
   `BLOCKED`, `UNSTABLE`, …) as a cross-check. [Q1][Q3]
5. **Fine-grained PATs cannot call the REST Checks API** (`/check-runs`, `/check-suites`). CI status
   therefore has to come from GraphQL `statusCheckRollup`, `gh pr checks`, or the Actions jobs API
   (which fine-grained PATs *can* use with "Actions: read"). [T1][T2]
6. **Failure logs.** `GET /repos/{o}/{r}/actions/jobs/{job_id}/logs` returns a 302 to a plain-text log
   that expires after 1 minute. `gh run view --log-failed` prints only the failed steps.
   `POST .../actions/runs/{run_id}/rerun-failed-jobs` exists if sous-chef ever needs to re-run jobs. [A1][A2][GH4]
7. **Draft PRs.** REST can *create* a draft (`draft: true`) but cannot flip it. Use the GraphQL
   mutations `markPullRequestReadyForReview` / `convertPullRequestToDraft`, or `gh pr ready`. [PR1][Q1]
8. **Linking.** `Closes #n` in the PR body links and auto-closes the issue only when the PR targets the
   **default branch**. Read the link back via `PullRequest.closingIssuesReferences` /
   `Issue.closedByPullRequestsReferences`. [L1][Q1][Q2]
9. **Tooling.** `octokit` (v4+, ESM) bundles REST, GraphQL, pagination, and default-on throttling
   and retry. `gh` already holds an OAuth token (`gh auth token` / `GH_TOKEN`), and `gh api graphql
   --paginate` handles cursors. For a personal CLI, the simplest auth is to reuse the user's
   `gh`/classic token (adding `project` scope if Projects is used). A fine-grained PAT is the
   least-privilege option, but it blocks Checks REST and user-owned Projects. A GitHub App is
   overkill until webhooks. [O1][O2][GH1][GH2][T1][AP1]
10. **Webhooks (later).** Every event sous-chef needs exists at repository level: `issues`,
    `pull_request`, `pull_request_review`, `pull_request_review_comment`, `pull_request_review_thread`
    (`resolved`/`unresolved`), `check_run`/`check_suite` (repository hooks get only
    `created`/`completed`), `workflow_run`, and `workflow_job`. The one exception is
    **`projects_v2_item`, which is organization-only** (public preview). GitHub does not
    auto-redeliver failed deliveries. [W1][W2]

---

## 1. Polling for Ready issues

### List and search filters

- `GET /repos/{owner}/{repo}/issues` filters on `state` (`open` default / `closed` / `all`),
  `labels` (comma-separated, all must match), `assignee` (login, `none`, `*`), `milestone`, `type`,
  `creator`, `mentioned`, `sort` (`created`/`updated`/`comments`), `direction`, and `since`. `since`
  is ISO 8601 and means "updated after the given time". `per_page` goes up to 100. [R2]
- **PRs show up in issue lists.** "GitHub's REST API considers every pull request an issue, but not
  every issue is a pull request". Filter them out by the presence of the `pull_request` key. [R2]
- `GET /search/issues?q=...` returns at most 1,000 results per search and is rate-limited to
  **30 requests/minute** (authenticated) for all search endpoints except code search. With a GitHub
  App *user access token*, `q` must include `is:issue` or `is:pull-request`, or the call returns 422. [S1]
- Useful search qualifiers: `is:issue is:open label:ready no:assignee`, `updated:>YYYY-MM-DD`, and
  `project:OWNER/NUMBER` (project number, not name). The search docs list no qualifier for a project
  *field* such as Status. [S2]

Inference: for one repo, `GET /repos/{o}/{r}/issues?labels=ready&state=open&assignee=none&sort=updated`
with an ETag is the cheapest Ready poll, since it avoids search's 30/min budget entirely.

### Labels vs Projects v2 Status as the Ready signal

| | Label `ready` | Projects v2 `Status = Ready` |
|---|---|---|
| Read | REST list or search filter [R2][S2] | GraphQL `ProjectV2.items(query: "status:Ready")` (filter syntax as in the project UI: `is:issue is:open status:Ready no:assignee`) [P2][P3], or REST `GET /users/{u}/projectsV2/{n}/items?q=...` [P4] |
| Write | `PATCH .../issues/{n}` `labels` (replaces the set), or add/remove label endpoints [R2] | `updateProjectV2ItemFieldValue` with `singleSelectOptionId`. Field and option IDs must be looked up first. You "cannot add and update an item in the same call" [P1] |
| Token | Fine-grained "Issues" read/write [T2] | Classic/OAuth `read:project` / `project` scope [P1]. **Fine-grained PATs cannot access user-owned Projects** [T1] |
| Webhook (future) | `issues.labeled` / `unlabeled`, repo-level [W1] | `projects_v2_item`, **org webhooks only**, public preview [W1] |
| Per-item read | `fieldValueByName(name: "Status")` on `ProjectV2Item` returns a union. Use a fragment on `ProjectV2ItemFieldSingleSelectValue { name optionId }` [P2] | |

Inference: on a **user-owned** repo, labels are much simpler. A Projects Status signal forces a
classic-scope token, GraphQL or the Projects REST API, and (later) has no repo-level webhook.

### Conditional requests and rate limits

- Send the saved `etag` back as `if-none-match` (or `last-modified` as `if-modified-since`). An
  unchanged resource returns `304`. "Making a conditional request does not count against your
  primary rate limit if a `304` response is returned" and the request was authorized with an
  `Authorization` header. Honour `x-poll-interval` if present. [B1]
- Primary limits: 5,000 REST requests/hour for a PAT or user token; 5,000 GraphQL points/hour.
  GitHub App installations start at 5,000 and scale up to 12,500. Actions' `GITHUB_TOKEN` gets
  1,000/hour/repo. [R1][G1]
- Secondary limits:
  - at most 100 concurrent requests (REST and GraphQL combined)
  - 900 points/minute for REST and 2,000 for GraphQL. GETs cost 1 point and mutations 5.
  - about 80 content-creating requests per minute and 500 per hour
  - 90 s of CPU time per 60 s of wall time [R1][G1]
- Guidance: make requests serially and wait at least 1 s between mutations. On 403/429, honour
  `retry-after` first, then `x-ratelimit-reset` when remaining is 0. Otherwise wait at least 1 minute,
  then back off exponentially. "Continuing to make requests while you are rate limited may result
  in the banning of your integration." [B1][R1]
- GraphQL connections need `first`/`last` between 1 and 100, and a single call can request at most
  500,000 nodes. Query `rateLimit { cost remaining resetAt }` to see a query's cost. [G1]

## 2. Writing back to issues

- **Labels and assignees:** `PATCH /repos/{o}/{r}/issues/{n}` with `labels` / `assignees`. Both
  *replace* the current set (`[]` clears them). Both need push access, and without it "label changes
  are silently dropped" (no error). `state` + `state_reason` (`completed`, `not_planned`, …) can be
  set too. [R2]
- **Comments:** `POST /repos/{o}/{r}/issues/{n}/comments` (`body`), and
  `PATCH /repos/{o}/{r}/issues/comments/{id}` to edit. Creating comments triggers notifications and
  can hit secondary limits. Use `since` (updated-after) when listing. [R3]
- **Projects v2:** `addProjectV2ItemById(projectId, contentId)` (idempotent: returns the existing
  item ID) and then `updateProjectV2ItemFieldValue`. [P1]
- **Linking PRs:**
  - The closing keywords are close/closes/closed, fix/fixes/fixed, and resolve/resolves/resolved,
    case-insensitive, with an optional colon. Cross-repo form: `Fixes owner/repo#100`. They work
    only when the PR targets the default branch ("these keywords are ignored, no links are
    created" otherwise). [L1]
  - Up to ten issues can be linked manually per PR through the Development sidebar. Keyword links
    are removed only by editing the PR body. [L1]
  - GraphQL reads: `PullRequest.closingIssuesReferences`, `Issue.closedByPullRequestsReferences`,
    and `CrossReferencedEvent.willCloseTarget`. `createLinkedBranch(issueId, oid, name)` creates a
    branch that shows under the issue's Development section (`Issue.linkedBranches`). [Q1][Q2]

  Inference: sous-chef can write `Closes #n` into the PR body for linking and use
  `createLinkedBranch` only if it wants the branch listed on the issue before the PR exists.

## 3. Pull requests

- **Create:** `POST /repos/{o}/{r}/pulls` with `title`, `head`, `base`, `body`, `draft`, and
  `maintainer_can_modify`. `issue` instead of `title` converts an existing issue into a PR. [PR1]
- **Update:** `PATCH .../pulls/{n}` accepts `title`, `body`, `state`, `base`, and
  `maintainer_can_modify`. It has **no `draft` field**. [PR1] To change draft status, use GraphQL
  `markPullRequestReadyForReview(pullRequestId)` / `convertPullRequestToDraft(pullRequestId)`. [Q1]
- **Mergeability:** `GET .../pulls/{n}` returns `mergeable`, which may be `null` while "GitHub has
  started a background job to compute the mergeability". Re-request after a delay. [PR1]
- **Request reviewers:**
  - `POST .../pulls/{n}/requested_reviewers` with `reviewers` (logins) and `team_reviewers`
    (slugs). It returns 422 if a reviewer isn't a collaborator. [RR1]
  - There's also `.../requested_reviewers/rerequest`, useful after pushing fixes. [RR1]
  - A reviewer who submits a review drops out of the requested list. [RR1]
  - GraphQL: `requestReviews(pullRequestId, userIds, teamIds, botIds, union)`. `union` adds to
    the existing set rather than replacing it. [Q1]

## 4. Review feedback

### What exists

| Thing | REST | GraphQL |
|---|---|---|
| Reviews (summary + state) | `GET .../pulls/{n}/reviews`, returned "in chronological order"; pending reviews have no `submitted_at` [RV1] | `reviews`, `latestReviews` (latest per user, excluding pending), `latestOpinionatedReviews`; `PullRequestReviewState` = APPROVED, CHANGES_REQUESTED, COMMENTED, DISMISSED, PENDING [Q1] |
| Inline review comments | `GET .../pulls/{n}/comments?since=` (updated after; sort `created`/`updated`); fields `in_reply_to_id`, `pull_request_review_id`, `line`/`original_line` [RC1] | `PullRequestReviewThread.comments` [Q1] |
| Threads + resolved state | **Not available.** The REST docs describe no resolution state [RC1] | `reviewThreads { isResolved isOutdated isCollapsed resolvedBy viewerCanResolve viewerCanReply path line subjectType comments }` [Q1] |
| Reply in a thread | `POST .../pulls/{n}/comments/{comment_id}/replies`, which must target the top-level comment ("Replies to replies are not supported") [RC1] | `addPullRequestReviewThreadReply(pullRequestReviewThreadId, body)` [Q1] |
| Resolve / unresolve | none | `resolveReviewThread(threadId)` / `unresolveReviewThread(threadId)` [Q1] |
| PR conversation comments | Issue comments endpoints ("Every pull request is an issue") [R3] | `PullRequest.comments` |
| Overall decision | none | `reviewDecision`: APPROVED / CHANGES_REQUESTED / REVIEW_REQUIRED [Q1] |

`addPullRequestReviewComment` is deprecated. Use `addPullRequestReviewThread` /
`addPullRequestReviewThreadReply` instead. [Q1]

### Detecting what's new since the last iteration

The docs have no "unread since X" primitive. Workable options:

- REST `since` on PR review comments and issue comments filters by *updated*-after, so it also
  catches edits. [RC1][R3]
- Track review IDs / `submitted_at` from the chronological reviews list. [RV1]
- Inference: the robust signal for "is there still something to fix" is the set of **unresolved,
  non-outdated threads** (GraphQL `reviewThreads` filtered on `isResolved == false`), plus any
  review with `CHANGES_REQUESTED` and any new conversation comments since the last pushed commit.
  Storing the last-seen comment/review IDs per Run covers the rest.
- Pagination caveat: Octokit's `graphql.paginate` supports one paginated resource per query, not
  nested pagination [O2]. Paging `reviewThreads` and then each thread's `comments` therefore needs
  separate queries (or `first: 100` with a check of `pageInfo`).

## 5. CI

- **Check runs** (GitHub Actions and other Apps):
  - `GET /repos/{o}/{r}/commits/{ref}/check-runs` with `filter=latest` (default), `status`, and
    `check_name`. [C1]
  - Statuses are `queued`, `in_progress`, `completed`, `waiting`, `requested`, `pending`. Conclusions
    are `success`, `failure`, `neutral`, `cancelled`, `skipped`, `timed_out`, `action_required`. [C1]
  - `output` has `title`, `summary`, `text`, and `annotations_count`. Annotations come from
    `GET .../check-runs/{id}/annotations` (`path`, `start_line`, `message`, …). [C1]
  - "Only GitHub Apps can write check runs"; users can read them. [C1]
  - **Not callable with fine-grained PATs** ("Call the Checks API" is a listed limitation, and no
    `/check-runs` endpoints appear in the fine-grained endpoint list). [T1][T3]
- **Commit statuses:** `GET .../commits/{ref}/status` gives a combined `state`: failure if any context
  is error or failure, pending if there are none or any is pending, success otherwise. It needs
  fine-grained "Commit statuses: read". [CS1][T2] (The REST page doesn't say whether check runs are
  included. Inference: they aren't, which is why the GraphQL rollup is preferable.)
- **GraphQL rollup:**
  - `Commit.statusCheckRollup` "Represents the rollup for both the check runs and status for a
    commit". Its `state` is a `StatusState` (ERROR, EXPECTED, FAILURE, PENDING, SUCCESS). [Q3]
  - Its `contexts` is a union of `CheckRun` (`name`, `status`, `conclusion`, `detailsUrl`,
    `isRequired(pullRequestNumber:)`, `checkSuite.workflowRun`) and `StatusContext`
    (`context`, `state`, `targetUrl`, `isRequired`). [Q3][Q4]
  - `PullRequest.statusCheckRollup` exists for the head ref. [Q1]
- **Logs:**
  - Find failed jobs with `GET .../actions/runs/{run_id}/jobs` (`filter=latest`). Each job has
    `steps[]` with `conclusion`, plus `check_run_url`. [A1]
  - Fetch the log with `GET .../actions/jobs/{job_id}/logs`: a 302 to a plain-text file, and "This
    link expires after 1 minute". It needs fine-grained "Actions: read". [A1][T2]
  - Whole-run logs come from `GET .../actions/runs/{run_id}/logs` (an archive). [A2]
  - `gh run view <run> --log-failed` shows only failed steps. `--exit-status` exits non-zero if
    the run failed. [GH4]
  - `gh pr checks` has `--required`, `--watch`, `--fail-fast`, and `--json bucket,state,name,link,workflow,…`,
    and exits with code 8 while checks are pending. [GH5]
- **"Approved + green" detector.** Inference from [Q1][Q3]: one GraphQL query on the PR:
  `isDraft`, `reviewDecision`, `mergeStateStatus`, and `commits(last:1){nodes{commit{oid statusCheckRollup{state}}}}`.
  - Done = `reviewDecision == APPROVED && rollup.state == SUCCESS` (optionally `mergeStateStatus == CLEAN`).
  - `MergeStateStatus.CLEAN` = "Mergeable and passing commit status". `UNSTABLE` = "Mergeable with
    non-passing commit status". `BLOCKED`, `BEHIND`, `DIRTY` (conflicts) and `UNKNOWN` cover the rest. [Q1]
  - Caveat: `reviewDecision` is nullable. The docs don't say when it is null. Inference: when no
    review is required by branch protection, so sous-chef should also fall back on `latestReviews`.

## 6. `gh` CLI vs Octokit, and auth

### Client

- **Octokit (`octokit` package):**
  - It includes `octokit.rest.*` (generated from the OpenAPI spec), `octokit.request`,
    `octokit.paginate` / `.iterator`, `octokit.graphql`, and `octokit.graphql.paginate`. [O1]
  - Throttling (`@octokit/plugin-throttling`) and retry (`@octokit/plugin-retry`) are **on by
    default**: "requests are retried once and warnings are logged in case of hitting a rate or
    secondary rate limit". [O1]
  - It requires Node 18+ (native fetch) and `module`/`moduleResolution: node16` in tsconfig. [O1]
    v4.0.0: "package is now ESM". [O3]
  - The `App` class covers GitHub App auth and webhooks for later. [O1]
  - `graphql.paginate` needs a `$cursor` variable and `pageInfo { hasNextPage endCursor }`, and
    handles one resource per query. [O2]
- **`gh` CLI:**
  - `gh api` covers any REST or GraphQL call, with `--paginate` (GraphQL needs `$endCursor` +
    `pageInfo`), `--slurp`, `--jq`, `--cache`, and `-H` for conditional headers. [GH1]
  - The high-level commands are `gh pr checks`, `gh pr view --json`, and `gh run view --log-failed`.
    `pr view --json` has `reviewDecision`, `statusCheckRollup`, `latestReviews`, `mergeStateStatus`,
    and `closingIssuesReferences`, but **no `reviewThreads`**. [GH3][GH4][GH5]
  - Auth comes from `gh auth login` or `GH_TOKEN`. [GH2]
- Inference: for a TypeScript tool, Octokit gives typed responses, built-in rate-limit handling,
  and no subprocess per call. `gh` is a good dev/debug aid and an auth source (`gh auth token`)
  but a poor primary transport (string parsing, a process spawned per call, no automatic
  conditional-request caching). The Coding Agent itself will likely use `gh` inside the worktree
  anyway.

### Auth for a personal CLI

- **Fine-grained PAT:**
  - It's the recommended token type: it covers one resource owner, can be limited to selected
    repos, and uses granular permissions. Expiry defaults to 30 days and can be set to 1–366 days
    or none. [T1]
  - **Limitations relevant here:** it cannot "Call the Checks API" and cannot "Access Projects owned
    by a user account" (it also can't access multiple orgs at once or Packages). [T1]
  - Permissions sous-chef would need: Issues (read/write), Pull requests (read/write), Commit
    statuses (read), Actions (read), and Contents (write, for pushing, presumably via git). [T2]
- **Classic PAT / gh OAuth token:**
  - It uses broad scopes. `gh`'s minimum is `repo`, `read:org`, `gist`, and Projects need
    `project`/`read:project`. [GH2][P1]
  - The current local `gh` login has `gist, read:org, repo, workflow`, so no project scope yet
    (observed via `gh auth status`).
  - Classic tokens unused for a year are removed. [T1]
- **GitHub App:**
  - It gets short-lived installation tokens ("will expire after 1 hour") minted from a JWT signed
    with the app's private key. [AP2]
  - It brings a bot identity, built-in webhooks, and scaling rate limits. [AP1]
  - GitHub's own guidance: PATs suit testing and short-lived scripts; build an App for long-lived
    integrations or organization access. [AP1]
- Inference: for v1 (polling, single developer, user-owned repo), use a token the user already has:
  `GH_TOKEN` or `gh auth token`, a classic/OAuth token with `repo` (plus `project` if Projects is
  the Ready signal). Support a fine-grained PAT as an option, but fall back to GraphQL
  `statusCheckRollup` for CI, which avoids the Checks REST limitation. That fallback is unverified:
  the docs don't say whether GraphQL check-run data is readable with a fine-grained PAT, so test it.
  A GitHub App earns its keep only with webhooks or when review comments should come from a bot
  rather than from Derek's account.

## 7. Webhooks (future seam, brief)

All of these are available as repository webhooks [W1]:

- `issues`: `labeled`, `unlabeled`, `assigned`, `opened`, `closed`, `edited`, …
- `issue_comment`: `created`, `edited`, `deleted`
- `pull_request`: `opened`, `synchronize`, `ready_for_review`, `converted_to_draft`,
  `review_requested`, `closed` (check `merged`), …
- `pull_request_review`: `submitted`, `edited`, `dismissed`
- `pull_request_review_comment`: `created`, `edited`, `deleted`
- `pull_request_review_thread`: `resolved`, `unresolved`
- `check_run` and `check_suite`: repo and org hooks receive only `created`/`completed` and
  `completed` respectively
- `status`
- `workflow_run` and `workflow_job`

Two exceptions and caveats:

- `projects_v2_item` is **organization webhooks only** and in public preview. User-owned Projects
  give you no item webhooks. [W1]
- Payloads are capped at 25 MB (larger events are not delivered). [W1] "GitHub does not
  automatically redeliver failed deliveries", but redelivery APIs exist for repo, org, and App
  hooks. [W2]
- Inference: a local CLI would need a tunnel (or `gh webhook forward`, not researched here) to
  receive webhooks. Polling stays as the reconciliation path either way.

## Open questions / not verified

- Whether GraphQL `statusCheckRollup` / `CheckRun` data is readable with a fine-grained PAT. The
  docs list "Checks API" as unsupported without saying whether that covers GraphQL. Test it with a
  real token.
- When exactly `reviewDecision` is `null`. The docs are silent.
- Fine-grained PAT permission needed for the Projects REST API (moot for user-owned projects, which
  fine-grained PATs can't reach).
- `mergeable_state` (REST) values aren't enumerated in the REST docs. Use GraphQL `mergeStateStatus`.

## Sources

- [B1] Best practices for using the REST API: https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api
- [R1] Rate limits for the REST API: https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api
- [R2] REST Issues: https://docs.github.com/en/rest/issues/issues
- [R3] REST Issue comments: https://docs.github.com/en/rest/issues/comments
- [G1] GraphQL rate and node limits: https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api
- [S1] REST Search: https://docs.github.com/en/rest/search/search#search-issues-and-pull-requests
- [S2] Searching issues and pull requests: https://docs.github.com/en/search-github/searching-on-github/searching-issues-and-pull-requests
- [P1] Using the API to manage Projects: https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-api-to-manage-projects
- [P2] GraphQL reference, Projects: https://docs.github.com/en/graphql/reference/projects
- [P3] Filtering projects: https://docs.github.com/en/issues/planning-and-tracking-with-projects/customizing-views-in-your-project/filtering-projects
- [P4] REST Project items: https://docs.github.com/en/rest/projects/items
- [L1] Linking a pull request to an issue: https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue
- [PR1] REST Pulls: https://docs.github.com/en/rest/pulls/pulls
- [RR1] REST Review requests: https://docs.github.com/en/rest/pulls/review-requests
- [RV1] REST Reviews: https://docs.github.com/en/rest/pulls/reviews
- [RC1] REST Review comments: https://docs.github.com/en/rest/pulls/comments
- [Q1] GraphQL reference, Pull requests: https://docs.github.com/en/graphql/reference/pulls
- [Q2] GraphQL reference, Issues: https://docs.github.com/en/graphql/reference/issues
- [Q3] GraphQL reference, Commits (StatusCheckRollup, StatusContext, StatusState): https://docs.github.com/en/graphql/reference/commits
- [Q4] GraphQL reference, Checks (CheckRun, CheckSuite, enums): https://docs.github.com/en/graphql/reference/checks
- [C1] REST Check runs: https://docs.github.com/en/rest/checks/runs
- [CS1] REST Commit statuses: https://docs.github.com/en/rest/commits/statuses
- [A1] REST Workflow jobs: https://docs.github.com/en/rest/actions/workflow-jobs
- [A2] REST Workflow runs: https://docs.github.com/en/rest/actions/workflow-runs
- [T1] Managing personal access tokens: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- [T2] Permissions required for fine-grained PATs: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
- [T3] Endpoints available for fine-grained PATs: https://docs.github.com/en/rest/authentication/endpoints-available-for-fine-grained-personal-access-tokens
- [AP1] Deciding when to build a GitHub App: https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/deciding-when-to-build-a-github-app
- [AP2] Generating an installation access token: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app
- [W1] Webhook events and payloads: https://docs.github.com/en/webhooks/webhook-events-and-payloads
- [W2] Handling failed webhook deliveries: https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries
- [O1] octokit.js README: https://github.com/octokit/octokit.js
- [O2] @octokit/plugin-paginate-graphql README: https://github.com/octokit/plugin-paginate-graphql.js
- [O3] octokit v4.0.0 release notes: https://github.com/octokit/octokit.js/releases/tag/v4.0.0
- [GH1] gh api: https://cli.github.com/manual/gh_api
- [GH2] gh auth login: https://cli.github.com/manual/gh_auth_login
- [GH3] gh pr view: https://cli.github.com/manual/gh_pr_view
- [GH4] gh run view: https://cli.github.com/manual/gh_run_view
- [GH5] gh pr checks: https://cli.github.com/manual/gh_pr_checks
