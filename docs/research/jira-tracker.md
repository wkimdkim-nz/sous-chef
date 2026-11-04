# Research: Jira as a Tracker

Question (issue #4): what does Jira offer sous-chef as a Tracker? Jira Cloud comes first, with Data Center (DC) differences noted.

Researched 2026-11-04. Cloud facts come mainly from the Jira Cloud platform REST API v3 OpenAPI spec as published that day (`swagger-v3.v3.json`, build `1001.0.0-SNAPSHOT-f6ec623d…`), plus Atlassian developer and support docs. DC facts come from the Jira 9.17.0 DC REST reference and DC admin docs. Sources are numbered at the end and cited as [n].

## TL;DR for sous-chef

1. **Auth:** an Atlassian **API token with HTTP Basic** (`email:token`) is the simplest way to run a personal, unattended CLI. Tokens now expire after at most 365 days (1 year by default). Scoped tokens are recommended, but they must be called through `https://api.atlassian.com/ex/jira/{cloudId}` rather than `https://<site>.atlassian.net` [4][5]. OAuth 2.0 (3LO) works but adds a lot of overhead: a browser consent step, rotating refresh tokens that expire after 90 days of inactivity, and a gateway base URL [6]. On DC, use a **Personal Access Token** as `Authorization: Bearer` [13].
2. **Polling:** use **`GET`/`POST /rest/api/3/search/jql`** (the "enhanced search"). The old `/rest/api/{2,3}/search` is marked deprecated and "Currently being removed" (CHANGE-2046) [1][2]. The new endpoint uses `nextPageToken` pagination and returns no `total`. It returns only `id` unless you pass `fields`, requires a *bounded* JQL query, and is **eventually consistent** [1][3].
3. **Rate limits:** API-token traffic is exempt from the new points-based hourly quotas, which have been enforced since 2026-03-02 for OAuth, Forge and Connect apps. Token traffic is still subject to per-endpoint **burst limits** (around 100 GET/s), and every caller is subject to **per-issue write limits** (20 writes/2 s, 100 writes/30 s). On a 429, honour `Retry-After` [7].
4. **Ready** can be expressed as any JQL, such as `status`, `statusCategory`, `labels`, `assignee`, `sprint` or a custom field `cf[NNNNN]`. Board columns are not JQL-queryable directly, because they map to statuses. The config's JQL filter *is* the definition of Ready.
5. **Writing back:** discover valid transitions per issue with `GET …/transitions`, then `POST …/transitions` with a transition **id**. Transition ids and names differ between workflows and projects. Comments are **ADF** in v3. Assignment uses `accountId` on Cloud and `name` on DC. Labels are edited with `PUT /issue` and `update.labels[{add|remove}]` [1].
6. **Linking PRs:** the only thing sous-chef can write with a plain token is a **remote issue link** (`POST …/remotelink`, idempotent by `globalId`). The **development panel** is populated by the GitHub for Atlassian app, which picks up the issue key in the branch name, commit messages or PR title. Its write API (`/rest/devinfo/0.10/bulk`) is only for Connect, Forge or OAuth apps [10][11][12]. The cheap path is to put the key in the branch name and PR title.
7. **Reading content:** in v3, the description, environment, comment bodies and textarea custom fields are **ADF** JSON. Atlassian provides no official REST endpoint that converts ADF to Markdown. Two workable paths: (a) walk the ADF tree yourself, or (b) request `expand=renderedFields`, which returns HTML, and convert that HTML to Markdown. The v2 API, and DC, use **wiki markup** strings instead [1][9][14].
8. **Webhooks:** Jira needs a **public HTTPS** endpoint. A personal user can't self-register a webhook with an API token: admin webhooks need a Jira admin, and dynamic webhooks need an OAuth 3LO app (5 per user per tenant, 30-day expiry with a refresh endpoint). Delivery is at-least-once, with up to 5 retries [8]. The design should keep polling as the baseline.
9. **DC is end-of-life on 2029-03-28.** New DC subscriptions ended for new customers on 2026-03-30 [16]. DC support should be treated as "nice to have": REST v2 only, `startAt`/`total` pagination, wiki markup, PAT bearer auth, and admin-configured rate limits [13][14][15].

---

## 1. Auth for a personal, unattended CLI

### Cloud: API token + Basic auth (recommended)
- Send `Authorization: Basic base64(email:api_token)` against `https://<site>.atlassian.net/rest/api/3/...`. Password auth is deprecated. The call gets the same permissions as that user has in the web UI [4].
- Atlassian describes Basic auth as suitable for "simple scripts and manual calls". It also says that *distributed apps* which collect users' API tokens don't meet its security requirements [4]. That rules out collecting other people's tokens, but it fits a personal CLI where the owner configures their own token.
- Repeated failed logins trigger a CAPTCHA, which blocks REST auth; the response carries the `X-Seraph-LoginReason: AUTHENTICATION_DENIED` header [4]. sous-chef should fail fast on 401 and not retry.
- **Expiry:** a token's lifetime is configurable from 1 to 365 days and defaults to one year. Older tokens, created before 2024-12-15, were set to expire between 2026-03-14 and 2026-05-12 [5]. **Implication:** sous-chef has to handle token expiry gracefully, with a clear 401 message and possibly a warning before expiry.
- **Scoped tokens** (recommended by Atlassian) must use the gateway: `https://api.atlassian.com/ex/jira/{cloudId}/rest/api/3/...` [5]. Unscoped tokens still work against the site URL. The docs give no deprecation notice for unscoped tokens [5]. The scopes sous-chef would need are `read:jira-work` and `write:jira-work`. Remote links, transitions, comments and assignment all fall under `write:jira-work` [1].
- **Service accounts:** an org admin can create an API token for a service account under Directory > Service accounts > Create credential > API token, selecting scopes as part of creation [5]. This is a good fit if the user wants sous-chef's comments to appear under a bot identity rather than their own.
- **cloudId** is needed for the gateway URL. With OAuth it comes from `GET https://api.atlassian.com/oauth/token/accessible-resources` [6]. (Inference, unverified: `https://<site>.atlassian.net/_edge/tenant_info` also returns it unauthenticated. It isn't in the docs I read.)

### Cloud: OAuth 2.0 (3LO)
- Authorization code grant only. Add the `offline_access` scope to get a refresh token. Refresh tokens **rotate**: every use issues a new one and disables the old. They expire after **90 days of inactivity**, with a 10-minute reuse leeway [6].
- All calls go to `https://api.atlassian.com/ex/jira/{cloudid}/...` [6].
- You have to create the app in the developer console with a callback URL, and apps are private unless you enable sharing [6]. 3LO is the only token type that can register **dynamic webhooks** and write **development information** [8][10].
- 3LO traffic is subject to the new points-based quota, which API-token traffic is not [7].

### Data Center: Personal Access Tokens
- Available since Jira 8.14. Send `Authorization: Bearer <PAT>`. Each user can create their own PATs in their profile, or via `POST /rest/pat/latest/tokens` [13].
- Expiry is optional. Admins can force a maximum lifetime (`atlassian.pats.max.tokens.expiry.days`, default 365), disable non-expiring tokens, or turn PATs off entirely. The default limit is 10 tokens per user [13].

## 2. Polling for Ready Work Items

### Endpoint: enhanced JQL search
- **`GET /rest/api/3/search/jql`** and **`POST /rest/api/3/search/jql`** are current. **`GET` and `POST /rest/api/3/search`** are flagged `deprecated: true`, with the summary "Currently being removed" and a link to changelog **CHANGE-2046** [1]. The same applies to v2 (`/rest/api/2/search/jql` exists) [1]. Community reports describe the old endpoint returning **HTTP 410 Gone** on Cloud from 2025 onward [2]. That is secondary evidence; the spec is authoritative that it's being removed.
- Parameters [1]:
  - `jql` must be **bounded**, meaning it has a search restriction. `order by key desc` on its own is rejected. `ORDER BY` can contain at most 7 fields.
  - `nextPageToken` is null for the first page and is omitted from the last page. The response also has `isLast`. The token **expires after 7 days**.
  - `maxResults` defaults to 50. The API may return fewer results than requested when many fields are requested. The most results per page (up to 5000) come back when requesting only `id` or `key`.
  - `fields` **defaults to `id` only**, unlike GET issue. Pass e.g. `summary,status,labels,assignee,updated,description` explicitly.
  - `expand` accepts `renderedFields`, `names`, `schema`, `transitions`, `operations`, `editmeta`, `changelog` and `versionedRepresentations`. `expand=transitions` returns the valid transitions inline.
  - `properties` returns up to 5 issue properties. Other parameters: `fieldsByKeys`, `failFast`, `includeArchivedProjects` (default false).
  - `reconcileIssues` accepts up to 50 issue ids for read-after-write consistency.
- **No `total`.** If a count is needed, use `POST /rest/api/3/search/approximate-count`, which is also eventually consistent and requires bounded JQL [1].
- **Eventual consistency:** "The API doesn't provide read-after-write consistency by default." Results can lag by seconds to minutes; bulk edits can take longer. `reconcileIssues` guarantees freshness only for the ids listed [3]. **Implication:** after sous-chef transitions an issue out of Ready, the next poll may still return it. sous-chef either passes recently touched ids in `reconcileIssues`, or (simpler) keeps its own local record of which Work Items have an active Run and ignores those.
- Helpers:
  - `POST /rest/api/3/jql/match` checks up to 50 issue ids against up to 10 JQL queries. It can re-confirm "still Ready?" right before starting a Run [1].
  - `POST /rest/api/3/issue/bulkfetch` fetches up to 100 issues per call, or up to 1000 with explicit, single-valued fields [1].
  - `POST /rest/api/3/changelog/bulkfetch` fetches changelogs in bulk [1].
- Since 2026-10-08, queries that use custom JQL functions return an immediate 400 with details from `/search/jql` [2a].

### `updated` cursors
- JQL date literals accept `"yyyy/MM/dd HH:mm"`, `"yyyy-MM-dd HH:mm"`, `"yyyy/MM/dd"` and `"yyyy-MM-dd"`, or relative values such as `"-5m"`. **The finest precision is minutes**, and dates are evaluated in **the calling user's configured time zone** [9].
- **Implication:** a cursor of the form `updated >= "<last poll minus a margin>"` needs overlap and de-duplication by issue id plus `updated` timestamp. Eventual consistency makes the overlap mandatory, not optional. Because Ready sets are small, it's simpler to **re-run the full Ready JQL every poll** and diff it against local state. A cursor is only an optimization.

### Rate limits (Cloud) [7]
- Three independent limiters:
  - **Points-based hourly quota.** A request costs 1 point plus the cost of the objects returned; writes cost 1. The global pool is 65k points/hour, and per-tenant tiers apply for paid editions. Quotas reset at the top of each UTC hour.
  - **Per-endpoint burst limits.** Steady state is GET 100/s, POST 100/s, PUT/DELETE 50/s; GET issue is allowed 150/s.
  - **Per-issue write limits:** 20 writes per 2 s and 100 writes per 30 s.
- Points-based enforcement began **2026-03-02** for Forge, Connect and OAuth 3LO apps. Atlassian states: "API token-based traffic is not affected by this change, and will continue to be governed by existing burst rate limits" [7].
- Headers:
  - `Retry-After` on 429, and sometimes on 503.
  - `X-RateLimit-Limit`, `-Remaining` and `-Reset` (the last is sent on 429 only).
  - `X-RateLimit-NearLimit`.
  - `RateLimit-Reason`, which is one of `jira-quota-global-based`, `jira-quota-tenant-based`, `jira-burst-based` or `jira-per-issue-on-write`.
  - The `Beta-RateLimit*` headers are informational only.
- Recommended retry behaviour: exponential backoff with jitter (start around 2 s, ×2 per retry, ×0.7–1.3 jitter, about 4 retries max), and retry only idempotent requests [7].
- For sous-chef, polling a single JQL every 30–60 s is far below these limits.

### Data Center differences
- `GET`/`POST /rest/api/2/search` uses classic offset pagination: `startAt`, `maxResults` and `total`. The DC docs say `total` should be treated as optional. `maxResults` defaults to 50, and the server caps it with `jira.search.views.default.max`. `fields` defaults to all navigable fields, and `validateQuery` defaults to true [14]. DC has no `/search/jql` or `nextPageToken`. (Inference: DC search reads from a local Lucene/OpenSearch index and is effectively consistent after indexing; I didn't verify this.)
- DC rate limiting is **off unless an admin enables it**. It uses a per-user token bucket, applied per node in a cluster, configured under Administration > System > Rate limiting, and only external REST requests are limited. Over-limit requests get a 429 [15].

## 3. Ways to express Ready

Everything comes down to the JQL filter in the sous-chef config [9]:

| Signal | JQL example | Notes |
|---|---|---|
| Status | `status = "Ready for Dev"` | Most natural fit. Status names differ per workflow. `statusCategory = "To Do"` is coarser and workflow-independent. |
| Label | `labels = sous-chef` | Easiest for a user to toggle, and doesn't need a workflow change. sous-chef can remove the label on pickup (a cheap way to claim the Work Item). |
| Custom field | `cf[10050] = "Yes"` or `"Agent Ready" = Yes` | Needs admin setup. The field id is site-specific. |
| Sprint | `sprint in openSprints()` | Combine it with another signal. |
| Board column | (no direct JQL) | Columns map to statuses, so use the status. A board's own filter can be reused via `filter = <id>`. |
| Assignee | `assignee = "<bot accountId>"` or `assignee = currentUser()` | Good when using a service-account identity. On Cloud, prefer `accountId` to names. |

Always add a project clause, e.g. `project = ABC AND …`. It keeps the query *bounded*, which `/search/jql` requires [1], and keeps one config scoped to one repo.

## 4. Writing back

- **Transitions** [1]:
  - `GET /rest/api/3/issue/{key}/transitions` lists the transitions available from the issue's current status, for the calling user. Its parameters are `transitionId`, `expand=transitions.fields`, `skipRemoteOnlyCondition`, `includeUnavailableTransitions` and `sortByOpsBarAndStatus`.
  - Without the *Transition issues* permission, the list is empty.
  - `POST /rest/api/3/issue/{key}/transitions` takes `{"transition":{"id":"…"}}`, plus `fields` and `update` for any fields on the transition screen.
  - `PUT /issue` ignores status changes; transitions are the only way to move an issue.
  - **Implication:** the config should name a *target status*, e.g. "In Progress" or "In Review". sous-chef then looks up the transition whose `to.name` matches it, rather than hard-coding transition ids, which differ by workflow. Required fields on a transition screen can make the call fail, so sous-chef needs a clear error for that case.
- **Comments:** `POST /rest/api/3/issue/{key}/comment`. In v3 the body is **ADF**. v2 and DC accept a wiki-markup string [1][14]. Visibility can be restricted to a role or group.
- **Assignment:** `PUT /rest/api/3/issue/{key}/assignee` with `{"accountId": "…"}`. `null` unassigns the issue and `"-1"` assigns the project default. This only needs the *Assign issues* permission, whereas a general edit needs *Edit issues* [1]. On DC, assignment uses `name` [14].
- **Labels and other fields:** `PUT /rest/api/3/issue/{key}` with `{"update":{"labels":[{"add":"x"},{"remove":"y"}]}}`. Pass `returnIssue=true` to get the updated issue back, e.g. to feed `reconcileIssues` [1][3].
- **History:** `GET /rest/api/3/issue/{key}/changelog` is paginated, oldest first [1]. sous-chef can use it to detect a human moving the Work Item back to Ready after a Run.
- Per-issue write limits (section 2) cap how fast sous-chef can comment on a single issue. Batch progress updates into one comment, or edit an existing comment.

## 5. Linking to GitHub PRs

- **Remote issue links** are available to any auth type:
  - `POST /rest/api/3/issue/{key}/remotelink` with `globalId`, `application`, `relationship` and `object{url,title,summary,icon,status{resolved,icon}}`.
  - When a link with the same `globalId` exists, the call **updates** it, and fields missing from the request are set to null. `globalId` is at most 255 characters [1]. Uniqueness of `globalId` is enforced per issue, and a link marked `status.resolved=true` is shown struck through [12].
  - **Implication:** sous-chef can upsert one link per PR, e.g. `globalId = "github-pr:<owner>/<repo>#<n>"`. It can mark the link resolved when the PR merges, without storing Jira's link id.
  - Requires the *Link issues* permission and issue linking to be enabled on the site [1].
- **Development panel and the GitHub for Atlassian app:**
  - A Jira **site admin** installs the app, and a **GitHub org owner** connects the org. It supports GitHub Cloud and Enterprise Cloud; GitHub Enterprise Server is covered by a separate article [10].
  - The panel shows branches, commits and PRs whose **branch name, commit message or PR title contains the issue key** [10].
  - sous-chef only needs to put the key into the branch name and PR title, e.g. `ABC-123-…`. Once the app is installed, the panel fills in automatically.
- **Development Information API** (`POST /rest/devinfo/0.10/bulk`): this is how an integration *pushes* repository, commit, branch and PR entities itself. It needs a Connect app with the `jiraDevelopmentTool` module, a Forge app with `devops:developmentInfoProvider`, or OAuth with the `write:dev-info:jira` scope. Processing is asynchronous and returns 202 [11]. It isn't usable with an API token, and isn't worth it given the GitHub app.
- **Smart Commits** [12a]:
  - Syntax: `ABC-123 #comment …`, `#time 1h`, or `#<transition-name>`, where only the part of the name before the first space counts, so use `#start-review`.
  - The committer's email must match exactly one Jira user. Smart Commits must be enabled, and they work through the GitHub integration.
  - They fail silently when a transition has required fields, and Atlassian advises against putting them in PR titles.
  - **Implication:** sous-chef should drive transitions through REST, not Smart Commits. If anything, sous-chef should *avoid* accidentally writing `#word` tokens after an issue key in commit messages.

## 6. Reading content

- **ADF** (Atlassian Document Format) is versioned JSON: `{"version":1,"type":"doc","content":[…]}` [9a].
  - Block nodes: paragraph, heading, bulletList, orderedList, listItem, codeBlock, blockquote, panel, table/tableRow/tableCell/tableHeader, rule, expand/nestedExpand, mediaSingle/mediaGroup/media, taskList/blockTaskItem, sync blocks and extensions.
  - Inline nodes: text, hardBreak, mention, emoji, date, status, inlineCard and mediaInline.
  - Marks: strong, em, code, link, strike, underline, subsup, textColor and border [9a].
  - Atlassian notes that marks and nodes in the JSON schema "may not be valid in this implementation" [9a], so a converter must tolerate unknown nodes.
- **Where ADF appears** (v3): the comment `body`, and the `description` and `environment` rich-text fields [1]. Textarea custom fields also use ADF [9a]. v2 returns the same fields as wiki-markup strings, and so does DC, which has no v3 [1][14].
- **Converting to Markdown for the Coding Agent:** the REST API has no ADF→Markdown endpoint. Options:
  1. Write a small ADF→Markdown walker. ADF is a simple tree, and the Coding Agent tolerates lossy output: render panels as blockquotes, mentions as `@displayName`, media as attachment references, and pass unknown nodes through as their text. This needs no new dependency and is easy to test.
  2. Request `expand=renderedFields` [1], which returns HTML for the description and comments (and `renderedBody` on comments), then convert the HTML with an HTML→Markdown library. This route also works on DC and v2, so one code path covers both, and Jira does the rendering.
  - Inference, not verified: Atlassian publishes `@atlaskit/editor-markdown-transformer` on npm. It is aimed at the editor (Markdown→ProseMirror) and is probably too heavy for this.
- **Attachments:**
  - Request `fields=attachment` to get metadata (id, filename, mimeType, size, `content` URL).
  - Download with `GET /rest/api/3/attachment/content/{id}`, which supports `Range` and a `redirect` parameter; a thumbnail endpoint also exists [1].
  - The *Browse projects* permission applies, and attachments added in private comments inherit the comment's restriction [1].
  - Inference: ADF `media` nodes reference Media Services file ids, which are not Jira attachment ids. Matching inline images to attachments is easier by filename, or through the `renderedFields` HTML.

## 7. Webhooks (for the future webhook seam) [8]

- **Ways to register:**
  - Admin UI or admin REST (`/rest/webhooks/1.0/webhook`), which needs a Jira admin.
  - Connect or Forge app modules.
  - Workflow post functions.
  - **Dynamic webhooks** via `POST /rest/api/3/webhook`, available only to Connect and OAuth 2.0 apps [1][8].
- **Dynamic webhook limits:**
  - OAuth apps get **5 webhooks per app per user per tenant**; Connect apps get 100.
  - They **expire after 30 days** unless refreshed with `PUT /rest/api/3/webhook/refresh` [1][8].
  - For non-public OAuth apps, webhooks are delivered only if the app owner is the user who registered them [1].
- **JQL filter:** supported for issue, comment, worklog, attachment and property events. Dynamic webhooks support only a subset of clauses (`project`, `issuetype`, `status`, `assignee`, `issueKey`, `priority`, `reporter`, indexed property, epic `cf[id]`) and operators (`=`, `!=`, `IN`, `NOT IN`). **`labels` is not in that subset** [8].
- **Events** include `jira:issue_created`, `jira:issue_updated` (with a `changelog`), `jira:issue_deleted` and `comment_created/updated/deleted`. Every payload includes `webhookEvent`, `timestamp`, `user` and `issue` [8].
- **Delivery:**
  - **HTTPS only**, to a publicly trusted certificate on an allowed port, and the receiver must allow-list Atlassian's IPs.
  - At-least-once: up to 5 retries, 5–15 minutes apart, on 408/409/425/429/5xx. Deduplicate with `X-Atlassian-Webhook-Identifier`.
  - Target latency is 30 s for primary events and 15 min for secondary events from bulk operations.
  - Admin webhooks can have a secret and are signed with HMAC in `X-Hub-Signature` [8].
- **Implication for sous-chef:** a local CLI would need a tunnel plus either admin rights or a 3LO app. A webhook seam should feed `issue_updated` events into the same "re-check Ready" path that polling uses, with the event's issue id passed to `reconcileIssues` [3]. Polling stays the source of truth.

## 8. Data Center summary and lifecycle

| Aspect | Cloud | Data Center |
|---|---|---|
| Auth | API token (Basic), scoped token via gateway, OAuth 3LO | PAT (Bearer) [13], Basic still possible |
| Search | `/rest/api/3/search/jql`, `nextPageToken`, no total, eventually consistent | `/rest/api/2/search`, `startAt`/`total` [14] |
| Rich text | ADF (v3) or wiki markup (v2) | Wiki markup only (REST v2) [14] |
| Users | `accountId` | `name`/`key` |
| Rate limits | Burst limits, per-issue write limits, points quota for apps [7] | Optional, admin-set token bucket per user per node [15] |
| Webhooks | Admin, dynamic (3LO), app | Admin UI/REST, post functions |

- **DC end of life:** 2029-03-28. New customers could no longer buy DC from 2026-03-30, and existing customers can expand until 2028-03-30. After end of life, DC products go read-only [16].
- **Implication:** build for Cloud. If DC is wanted at all, keep the Tracker adapter's request layer thin enough that a DC variant only swaps the search pagination, the auth header, the rich-text renderer and user identifiers.

## Recent deprecations and changes to watch

- **CHANGE-2046:** `/rest/api/{2,3}/search` (GET/POST) is being removed in favour of `/search/jql` [1][2]. Community reports describe 410 responses.
- **API token expiry:** tokens default to and are capped at 1 year (since 2024-12-15). Pre-existing tokens expired between March and May 2026 [5].
- **Points-based rate limits** have been enforced since 2026-03-02 for OAuth, Forge and Connect apps; API tokens are exempt [7].
- **2026-10-08:** custom JQL functions are validated directly by search, with immediate 400s [2a].
- **`/rest/api/3/workflow/search`** is deprecated in favour of `/rest/api/3/workflows/search` [1]. This only matters if sous-chef inspects workflows.
- The Connect framework is being wound down in favour of Forge [8]. This is only relevant if sous-chef ever becomes an installed app.
- Data Center end of life is 2029-03-28 [16].

## Open questions and items not verified

- The exact removal dates in the CHANGE-2046 changelog entry. The changelog page didn't render for my fetcher; the deprecation status comes from the OpenAPI spec.
- The exact access-token lifetime for 3LO (the docs only give `expires_in`), and whether 3LO accepts a `localhost` redirect URI.
- Whether Cloud JQL `development[pullrequests]…` fields (which query dev-panel data) are usable for "already has a PR" checks.

## Sources

1. Jira Cloud platform REST API v3, OpenAPI spec (fetched 2026-11-04): https://developer.atlassian.com/cloud/jira/platform/swagger-v3.v3.json. Reference UI: https://developer.atlassian.com/cloud/jira/platform/rest/v3/ (issue search, issues, transitions, comments, remote links, attachments, webhooks, JQL match, bulk fetch). The v2 spec is at https://developer.atlassian.com/cloud/jira/platform/swagger.v3.json.
2. CHANGE-2046 notice, linked from the deprecated endpoints: https://developer.atlassian.com/changelog/#CHANGE-2046. Secondary corroboration of the 410 responses: https://github.com/atlassian/atlassian-mcp-server/issues/70 and https://community.atlassian.com/forums/Jira-questions/why-HTTP-410-Errors-from-rest-api-3-search-error/qaq-p/3109922
   - 2a. Jira Cloud platform changelog, 2026-10-08 "Direct JQL validation for custom JQL functions returns to Search REST APIs": https://developer.atlassian.com/cloud/jira/platform/changelog/
3. Search and reconcile: https://developer.atlassian.com/cloud/jira/platform/search-and-reconcile/
4. Basic auth for REST APIs: https://developer.atlassian.com/cloud/jira/platform/basic-auth-for-rest-apis/
5. Manage API tokens for your Atlassian account: https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/
6. OAuth 2.0 (3LO) apps: https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/
7. Rate limiting (Cloud): https://developer.atlassian.com/cloud/jira/platform/rate-limiting/
8. Webhooks (Cloud): https://developer.atlassian.com/cloud/jira/platform/webhooks/
9. JQL fields (Cloud): https://support.atlassian.com/jira-software-cloud/docs/jql-fields/
   - 9a. ADF structure: https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/
10. Integrate with GitHub (GitHub for Atlassian app): https://support.atlassian.com/jira-cloud-administration/docs/integrate-with-github/
11. Development Information API: https://developer.atlassian.com/cloud/jira/software/rest/api-group-development-information/
12. Using fields in remote issue links: https://developer.atlassian.com/server/jira/platform/using-fields-in-remote-issue-links/
    - 12a. Process work items with Smart Commits: https://support.atlassian.com/jira-software-cloud/docs/process-issues-with-smart-commits/
13. Using personal access tokens (DC): https://confluence.atlassian.com/enterprise/using-personal-access-tokens-1026032365.html
14. Jira DC REST API reference 9.17.0: https://docs.atlassian.com/software/jira/docs/api/REST/9.17.0/
15. Improving instance stability with rate limiting (DC): https://confluence.atlassian.com/adminjiraserver101/improving-instance-stability-with-rate-limiting-1455429187.html
16. Data Center end of life: https://www.atlassian.com/licensing/data-center-end-of-life
