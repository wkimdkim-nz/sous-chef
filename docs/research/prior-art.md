# Prior art: tracker-driven coding agents

Research for [#6](https://github.com/wkimdkim-nz/sous-chef/issues/6) (part of map #1). Researched 2026-11-04.

**Question.** What can sous-chef learn from existing tools that turn Tracker items into agent-authored PRs and keep iterating on feedback? For each tool: how Ready is signalled, the Run lifecycle and Tracker state changes, how review comments and CI failures get back to the agent, guardrails and limits, per-repo workflow customisation, and known failure modes.

**Method.** Primary sources only: official docs, READMEs, specs and source code. Each claim links to the source that owns it. Where a doc page didn't cover something, the cell says "not documented" instead of guessing. Vendor docs change quickly, so treat them as accurate on the research date.

Vocabulary: sous-chef terms (Tracker, Work Item, Ready, Coding Agent, Run) are used for sous-chef. Each tool's own terms ("issue", "session", "conversation") are kept when describing that tool.

---

## Short answer

The closest prior art is **OpenAI Symphony**, a published, language-agnostic spec for a long-running daemon. It polls a Tracker, gives each Work Item its own workspace, runs a Coding Agent there, and keeps the workflow in a repo-owned `WORKFLOW.md`. Its shape is almost exactly sous-chef's. The hosted products (Copilot coding agent, Cursor and Devin cloud agents, claude-code-action, OpenHands) agree on a set of guardrails: their own branch prefix, a human gate before merge, only trusted actors can trigger, bot comments are ignored by default, and attempts and time are capped.

**Patterns worth adopting**

1. **Reconcile against the Tracker on every poll, and keep no database of record.** Recover from a restart by re-reading the Tracker and existing worktrees (Symphony).
2. **Keep the workflow in the repo, versioned with the code.** Use one Markdown file: YAML front matter for config and a strict template body for the prompt. Fall back to a built-in default when it's missing, and validate it before every dispatch (Symphony `WORKFLOW.md`; similar to `sweep.yaml`, `.github/copilot-instructions.md`, `AGENTS.md`).
3. **Treat Ready as an edge (a transition into a state or label), not a level, and dedupe on it.** Use active and terminal state lists plus required labels (Symphony, Devin, OpenHands, Cursor).
4. **Give each Work Item one worktree and one branch under a fixed prefix.** Sanitise the key, keep the path inside a root, reuse the worktree across attempts, and delete it only when the Work Item is terminal (Symphony; `copilot/`, `claude/`, `openhands/issue-N`).
5. **Use one living status comment per Work Item, not a stream of comments** (Symphony "Codex Workpad", claude-code-action tracking comment, Copilot and Devin agent panels).
6. **Gather feedback from every channel and treat each actionable item as blocking.** The channels are top-level PR comments, inline review comments, review states and check runs. Each item is blocking until it is fixed or explicitly pushed back on. Repeat until no items remain and CI is green (Symphony feedback sweep).
7. **Bound everything:** turns per Run, attempts per feedback round, wall-clock and stall timeouts, CI polling duration, global concurrency, and exponential backoff on failure (Symphony, Sweep, Aider, Copilot).
8. **Trust boundary:** act only on Work Items and comments from users with write access. Ignore bot comments by default and allowlist known CI bots. Keep Tracker credentials out of the Coding Agent's environment.
9. **Stop on a Tracker state change.** Moving a Work Item out of an active state cancels its Run. This gives the user a "stop" lever with no extra UI (Symphony, Devin).

**Pitfalls to avoid**

1. **Bot-to-bot feedback loops** between the Coding Agent and review bots or CI bots (Devin documents this explicitly).
2. **Missing context that arrives after pickup.** Copilot doesn't see Work Item comments added after assignment. sous-chef should re-read the Work Item and PR on every iteration.
3. **Pushes that don't trigger CI.** GitHub Actions doesn't start workflows for events created with `GITHUB_TOKEN`. Copilot's workflows wait for approval, and Sweep's draft PRs skipped Actions. If sous-chef waits for CI that never starts, it hangs forever, so it needs a "no checks appeared" timeout.
4. **Waiting forever.** Agents that ask for user input, stalled sessions, and CI that never finishes all hang a Run. Treat input-required as a failure and detect stalls.
5. **Unbounded fix attempts** that burn money (Sweep caps at 10 edits, Aider at 3 reflections, Copilot at 59 minutes).
6. **Trusting an LLM's own success verdict.** The OpenHands resolver's `guess_success` asked an LLM whether it succeeded. Use deterministic signals instead: CI status, review state, and whether any comments remain unresolved.
7. **Prompt injection from Work Item and PR text.** Everyone warns about it. Sanitise hidden content and keep secrets and network access narrow.
8. **Fighting the human.** Rewriting PR history, opening a new PR per iteration, or merging. sous-chef already rules out merging; also push only to its own branch and keep one PR per Work Item.
9. **Coupling to a fast-moving vendor surface.** The OpenHands V0 resolver was removed in April 2026. SWE-agent is superseded by mini-swe-agent. Sweep's GitHub App was retired for a JetBrains plugin. Keep the Coding Agent and Tracker behind narrow adapters.

---

## Comparison at a glance

| Tool | Ready signal | Where it runs | Review feedback | CI feedback | Merges? | Per-repo config |
|---|---|---|---|---|---|---|
| OpenAI Symphony (spec) | Work Item in `active_states`, not in `terminal_states`, with all `required_labels` | Local daemon, one workspace per issue | Agent's feedback sweep in prompt, plus a human-set `Rework` state | Agent confirms checks green before handoff | Yes, via a `Merging` state the human sets | `WORKFLOW.md` (front matter + prompt) |
| GitHub Copilot coding (cloud) agent | Assign issue to Copilot (GitHub or Jira) | GitHub Actions / cloud sandbox | `@copilot` PR comments from write-access users | Agent Merge "Watch and fix" for failed workflows and merge conflicts | No | `.github/copilot-instructions.md`, `copilot-setup-steps.yml`, MCP |
| claude-code-action | `@claude` mention, assignee or label trigger | Your GitHub Actions runner | New `@claude` mention, which pushes to the open PR's branch | Can read Actions logs with `actions: read`; no automatic loop | No (cannot approve or merge) | `CLAUDE.md`, `prompt`, `claude_args` |
| Cursor cloud agents | Assign to Cursor or `@Cursor` in Linear; `@cursor` on GitHub; Linear triage rules | Cursor VM | Follow-up via `@Cursor`; Bugbot Autofix spawns agents for Bugbot findings | Not documented as automatic | No (human review before merge) | `.cursor/rules`, `AGENTS.md`, `.cursor/environment.json` |
| Devin | Assign in Linear, skill label, `@Devin` mention, or edge-triggered automation on team, label or status | Devin cloud VM | Auto-responds to comments on PRs its sessions own | Responds to CI-bot comments ("lint check failed" always processed) | Can, if allowed (docs recommend branch protection) | `AGENTS.md`, skills, playbooks, blueprints |
| OpenHands (Agent Canvas) | Trigger label or state (default `openhands`), polled every 15 min | Agent Canvas backend / Docker | Separate reviewer automation | Watchdog plus reviewer statuses | "Software factory" watchdog can | Agent profiles, repo microagents |
| OpenHands resolver (V0, removed Apr 2026) | `fix-me` label or `@openhands-agent` | GitHub Actions | `fix-me` on PR or `@openhands-agent` comment | Not documented | No (draft PR) | `.openhands/microagents/repo.md` |
| Sweep (retired) | Issue titled `Sweep:` or `Sweep` label | Sweep GitHub App | "Addresses developer replies & comments" | Polls check runs, feeds failing logs, at most 10 edits, 120-minute polling cap | No | `sweep.yaml` |
| SWE-agent | Manual CLI: `--problem_statement.github_url` | Local, Docker | None | None | No | Single YAML config |
| Aider | Manual CLI: `--message` | Local | None | `--auto-test`/`--auto-lint`, at most 3 reflections | No | `.aider.conf.yml`, conventions files |

---

## Per-tool findings

### OpenAI Symphony (closest analogue)

An Apache-2.0 spec plus an experimental Elixir reference implementation, published 2026-02. The README calls it "a low-key engineering preview for testing in trusted environments" ([README](https://github.com/openai/symphony/blob/main/README.md)). The spec states the same problem sous-chef has: "a long-running automation service that continuously reads work from a configured issue tracker, creates an isolated workspace for each issue, and runs a coding agent session for that issue inside the workspace" ([SPEC.md §1](https://github.com/openai/symphony/blob/main/SPEC.md)).

- **Ready signal.** An issue can be dispatched only if its state is in `tracker.active_states` and not in `terminal_states`, it carries every `required_labels` entry, the adapter marks it `dispatchable`, it isn't already claimed or running, and concurrency slots are free. Candidates are sorted by priority, then oldest `created_at`, then identifier ([§8.2](https://github.com/openai/symphony/blob/main/SPEC.md)). The polling default is `interval_ms: 30000` ([§5.3.2](https://github.com/openai/symphony/blob/main/SPEC.md)). The reference workflow uses Linear with active states `Todo, In Progress, Merging, Rework` and polls every 5 s ([elixir/WORKFLOW.md](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md)).
- **Run lifecycle.** The service keeps its own internal claim states (`Unclaimed → Claimed → Running / RetryQueued → Released`), separate from Tracker states. A run attempt has named phases and distinct terminal reasons: `Succeeded, Failed, TimedOut, Stalled, CanceledByReconciliation`. "Distinct terminal reasons are important because retry logic and logs differ" ([§7.1–7.2](https://github.com/openai/symphony/blob/main/SPEC.md)). After each turn the worker re-checks the Tracker and keeps going on the same thread while the issue is active, up to `max_turns` (default 20). Continuation turns send only continuation guidance, not the full prompt ([§7.1](https://github.com/openai/symphony/blob/main/SPEC.md)).
- **Tracker state changes.** These are deliberately *not* made by the orchestrator. "Ticket writes (state transitions, comments, PR links) are typically performed by the coding agent through provider-native tools". The tools run host-side so that "the child receives tool results, not a raw token" ([§1](https://github.com/openai/symphony/blob/main/SPEC.md), [§11.5](https://github.com/openai/symphony/blob/main/SPEC.md)). "A successful run can end at a workflow-defined handoff state (for example `Human Review`), not necessarily `Done`" ([§1](https://github.com/openai/symphony/blob/main/SPEC.md)). The reference status map is `Todo → In Progress → Human Review → (Rework | Merging) → Done` ([WORKFLOW.md](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md)).
- **Review and CI feedback.** This lives in the prompt, not the scheduler. The "PR feedback sweep protocol" collects top-level PR comments, inline review comments and review states. It treats "every actionable reviewer comment (human or bot)" as blocking until code changes or "explicit, justified pushback". It repeats until nothing is outstanding and "PR checks are passing (green)", and only then moves to `Human Review`. A human moving the issue to `Rework` triggers "a full approach reset": close the PR, start a fresh branch and plan ([WORKFLOW.md](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md)).
- **Reconciliation.** Every tick: (a) stall detection, which kills and retries a worker with no agent events for `stall_timeout_ms` (default 5 min); (b) refresh running issue states, stopping and cleaning up when terminal and stopping without cleanup when inactive. If the refresh fails, workers keep running ([§8.5](https://github.com/openai/symphony/blob/main/SPEC.md)). Restart recovery uses the Tracker and filesystem, "without requiring a persistent database" ([§2.1](https://github.com/openai/symphony/blob/main/SPEC.md), [§14.3](https://github.com/openai/symphony/blob/main/SPEC.md)).
- **Limits.** `max_concurrent_agents` (10) with optional per-state caps. Failure retry backoff is `min(10000 * 2^(attempt-1), max_retry_backoff_ms)` with a 5-minute cap. A clean exit gets a 1-second continuation re-check. `turn_timeout_ms` is 1 h and hook `timeout_ms` is 60 s ([§5.3, §8.3–8.4](https://github.com/openai/symphony/blob/main/SPEC.md)). Approval or user-input requests "MUST NOT leave a run stalled indefinitely"; the example high-trust policy treats user-input-required as a hard failure ([§10.5](https://github.com/openai/symphony/blob/main/SPEC.md)).
- **Workspace safety.** The agent's cwd must equal the per-issue workspace. The workspace must stay under the root. Keys are sanitised to `[A-Za-z0-9._-]` with a hash suffix. Workspaces are reused across runs and not deleted on success ([§9](https://github.com/openai/symphony/blob/main/SPEC.md)). Repo population (clone, deps) is done in `after_create`/`before_run` hooks ([§9.3–9.4](https://github.com/openai/symphony/blob/main/SPEC.md)).
- **Per-repo customisation.** The repo-owned `WORKFLOW.md` holds YAML front matter (tracker, polling, workspace, hooks, agent, codex) and a Liquid-style prompt body. "Unknown variables MUST fail rendering". Changes hot-reload, and an invalid reload keeps the last good config ([§5–6](https://github.com/openai/symphony/blob/main/SPEC.md)).
- **Failure modes it calls out.** Prompt injection leading to "data leaks, destructive mutations, or full machine compromise". Recommended mitigations: tighter sandboxing, filtering which Tracker sources are eligible, and narrowing tools and credentials ([§15.5](https://github.com/openai/symphony/blob/main/SPEC.md)). The SSH-worker appendix warns against re-executing the same issue on several hosts ([App. A.3](https://github.com/openai/symphony/blob/main/SPEC.md)).
- **Difference from sous-chef.** Symphony merges (a `land` skill when a human sets `Merging`). sous-chef never merges.

### GitHub Copilot coding agent ("Copilot cloud agent")

- **Ready signal.** Assigning an issue: "Assigning an issue always creates a pull request" ([kick off a task](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/kick-off-a-task)). There's also Jira: "Assign a Jira issue to GitHub Copilot (or @mention it in issue comments)" ([changelog 2026-03-05](https://github.blog/changelog/2026-03-05-github-copilot-coding-agent-for-jira-is-now-in-public-preview/)).
- **Lifecycle.** It "works on the task and requests your review when it finishes". It "receives the issue title, description, and existing comments at assignment time" and "does not see comments added after assignment, so post follow-up information on the pull request instead" ([kick off a task](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/kick-off-a-task)). For Jira it posts progress updates and clarifying questions in the Jira agent panel and opens a draft PR ([changelog](https://github.blog/changelog/2026-03-05-github-copilot-coding-agent-for-jira-is-now-in-public-preview/)). "Copilot can only work on one branch at a time and can open one pull request per task" ([about](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)).
- **Review feedback.** "Mention `@copilot` in comments on the pull request". Batch comments with "Start a review". "Copilot only responds to comments from people who have write access" ([best results](https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results)).
- **CI feedback.** Agent Merge "Watch and fix" "watches a pull request for blockers you choose and starts work automatically when it finds one". The blockers are failed Actions workflows and merge conflicts. It doesn't merge: "Enabling Agent Merge does not enable auto-merge" ([Agent Merge](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/get-pull-requests-ready-to-merge)).
- **Guardrails** ([risks and mitigations](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/risks-and-mitigations)):
  - Only write-access users can trigger it.
  - It pushes only to a `copilot/` branch.
  - It "cannot mark its pull requests as 'Ready for review' and cannot approve or merge".
  - Workflows don't run until a write-access user approves.
  - The requester can't approve the PR.
  - Internet access goes through a firewall.
  - Hidden characters are filtered from input.
  - Commits are authored by Copilot and co-authored by the assigner.
  - Sessions on Actions have "a maximum execution time of 59 minutes" that "cannot be extended" ([about](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)).
- **Per-repo customisation.** `.github/copilot-instructions.md` plus path-specific instruction files, `copilot-setup-steps.yml` for preinstalled dependencies, and MCP servers ([best results](https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results)).
- **Failure modes.** Output "can produce incorrect or suboptimal code, including code that contains security vulnerabilities" ([about](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)). The docs advise against giving it security-sensitive work or incident response ([best results](https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results)). It goes blind to Work Item comments made after pickup (above).

### claude-code-action (Anthropic)

- **Ready signal.** `trigger_phrase` (default `@claude`, matched on word boundaries), `assignee_trigger`, and `label_trigger` ([usage.md](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md), [faq.md](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)). The mode (interactive tag vs. automation with an explicit `prompt`) is detected automatically.
- **Lifecycle.** It "operates by updating a single initial comment with progress and results", with progress checkboxes. On an issue it always creates a new branch (prefix `claude/`). On an open PR it pushes to the PR branch. On a closed PR it makes a new branch ([capabilities-and-limitations.md](https://github.com/anthropics/claude-code-action/blob/main/docs/capabilities-and-limitations.md)). By default it doesn't open the PR: "The user must click the link and create the PR themselves" ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)).
- **Review and CI feedback.** No automatic loop. A new `@claude` mention starts a new run. With `actions: read` it "can access workflow runs, job logs, and test results" ([capabilities](https://github.com/anthropics/claude-code-action/blob/main/docs/capabilities-and-limitations.md), [faq.md](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)).
- **Guardrails.**
  - Write-access actors only. Bots are blocked unless listed in `allowed_bots`, and the docs warn that "Allowed bots are not checked for repository permissions" ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)).
  - It cannot approve PRs, submit formal reviews, merge or rebase ([capabilities](https://github.com/anthropics/claude-code-action/blob/main/docs/capabilities-and-limitations.md)).
  - Input is sanitised: HTML comments, invisible characters and hidden attributes are stripped.
  - On PRs, Claude config files (`CLAUDE.md`, `.claude/`, `.mcp.json`…) are restored from the base branch so a PR can't rewrite the agent's instructions ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)).
- **Per-repo customisation.** `CLAUDE.md`, workflow YAML `prompt`, and `claude_args` (for example `--max-turns`, `--allowedTools`) ([usage.md](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md)).
- **Failure modes.** "The `github-actions` user cannot trigger subsequent GitHub Actions workflows", so pushes and comments made with `GITHUB_TOKEN` don't start CI ([faq.md](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)). Shallow clones (`--depth=1` / `20`) surprise agents that need history ([faq.md](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)).

### Cursor cloud (background) agents + Linear / GitHub / Bugbot

- **Ready signal.** In Linear: assign to Cursor or `@Cursor` in a comment, or use Linear triage rules that "assign issues to Cursor, or start Cloud Agents based on conditions". Repo routing checks `[repo=owner/repo]` in the text, then issue labels, then project labels, then a default ([Linear integration](https://cursor.com/docs/integrations/linear)). On GitHub: comment `@cursor` on a PR or issue ([cloud agents](https://cursor.com/docs/cloud-agent)).
- **Lifecycle.** "Start, Provision, Run, Persist, Hand off, Recycle". The agent "pushes its branch and opens a pull request for a human to review before anything merges" ([security overview](https://cursor.com/docs/cloud-agent/security.md)). It posts real-time status to Linear ([Linear](https://cursor.com/docs/integrations/linear)).
- **Feedback.** Follow-ups go through `@Cursor` in Linear or the agent session. Bugbot reviews each PR update, and "Bugbot Autofix automatically spawns a Cloud Agent to fix bugs found during PR reviews". Bugbot ignores manual-trigger commands posted by bots ([Bugbot](https://cursor.com/docs/bugbot.md)). Automatic CI-failure fixing is not documented.
- **Guardrails.** One Firecracker microVM per agent. Network egress allowlists. Redacted runtime secrets. Signed commits. An agent "can only reach repositories the triggering user could already reach" ([security overview](https://cursor.com/docs/cloud-agent/security.md)).
- **Per-repo customisation.** `.cursor/rules/*.mdc`, `AGENTS.md`, skills, and `.cursor/environment.json` ([best practices](https://cursor.com/docs/cloud-agent/best-practices.md)). Headless CLI: `agent -p` with `--force` to apply edits and `--output-format stream-json` for progress ([CLI headless](https://cursor.com/docs/cli/headless)).
- **Failure modes.** "Cloud Agents auto-run terminal commands", so prompt injection can try to exfiltrate code ([security overview](https://cursor.com/docs/cloud-agent/security.md)). "If it is hard for a human developer to test locally, it will also be hard for an agent" ([best practices](https://cursor.com/docs/cloud-agent/best-practices.md)).

### Devin (Cognition)

- **Ready signal.** In Linear: assign to Devin (uses the default skill), add a `@skills:<name>` label, or `@mention` Devin. Automation triggers filter by team, label and status, and they "use **edge detection**, meaning they only fire when a ticket transitions from not matching to matching" ([Linear](https://docs.devin.ai/integrations/linear.md)). On GitHub: a comment that begins with `/devin`, from write or admin users, on open PRs only ([GitHub](https://docs.devin.ai/integrations/gh.md)).
- **Lifecycle and Tracker updates.** Uses Linear's agent session UI: an activity feed, plan tracking synced to Linear's plan UI, follow-up messages, a stop signal ("put Devin to sleep"), and the PR link ([Linear](https://docs.devin.ai/integrations/linear.md)).
- **Review and CI feedback.** "Devin automatically responds to comments on PRs its sessions are working on, as long as the session has not been archived". A `/devin` comment on a PR that a session already owns goes to that session ([GitHub](https://docs.devin.ai/integrations/gh.md)). By default it ignores bot comments "to prevent infinite feedback loops". The options are No bots (default), All bots, or an allowlist. "Lint failure comments from bots (containing 'lint check failed') are always processed", and you can allowlist CI bots so it fixes test and build failures ([bot comment settings](https://docs.devin.ai/product-guides/bot-comment-settings.md)).
- **Guardrails.** The docs recommend branch protection "to ensure all required checks pass before Devin can merge changes" ([GitHub](https://docs.devin.ai/integrations/gh.md)). Repo access is scoped, and the GitHub App permissions include `checks`/`actions` to read and re-run CI.
- **Per-repo customisation.** `AGENTS.md` (16 KiB auto-inject limit), skills (`SKILL.md`), playbooks, environment blueprints, and a `devin_pr_template.md` PR template ([docs index](https://docs.devin.ai/llms.txt), [GitHub](https://docs.devin.ai/integrations/gh.md)).
- **Failure modes.** "All bots" mode "may cause infinite loops with automated code review bots" ([bot comment settings](https://docs.devin.ai/product-guides/bot-comment-settings.md)).

### OpenHands

- **Current: Agent Canvas "Issue to Pull Request" automation.** Supports GitHub Issues, GitLab, Jira and Linear.
  - Polls on a schedule (default every 15 min). Ready is a trigger label or state (default label `openhands`).
  - "Deduplicates: tracks which issues have already been processed using persistent state" and "caps parallel work".
  - Branch `openhands/issue-42`, PR title `[#42] <title>` with `Closes #42`, then a comment back on the Work Item. PRs are drafts by default.
  - To retry, "Remove the trigger label… Re-apply the trigger label".
  - Security: "Content is untrusted"; use agent profiles to limit secrets ([issue-to-PR](https://docs.openhands.dev/openhands/usage/agent-canvas/prebuilt/issue-to-pr.md)).
- **"Software factory."** Four independent cron automations: triage applies `ready-for-dev`; developer opens the PR; a reviewer runs tests and posts exact-head statuses; a deterministic watchdog merges only a current branch with passing statuses. Each role gets its own least-privilege token. When an accepted branch went stale, "the watchdog updated it and waited for a fresh exact-head review before merging" ([software factory](https://docs.openhands.dev/openhands/usage/use-cases/software-factory.md)).
- **Legacy V0 resolver (GitHub Action).**
  - Triggered by the `fix-me` label (considers the whole thread) or an `@openhands-agent` comment (only that comment).
  - It would "Create a draft PR if successful, or push a branch if unsuccessful", comment results, and "Remove the 'fix-me' label once processed".
  - Repo instructions came from `.openhands/microagents/repo.md` ([resolver README @0.40.0](https://github.com/OpenHands/OpenHands/blob/0.40.0/openhands/resolver/README.md)).
  - Success was judged by an LLM (`guess_success`, `_check_review_thread`…) ([issue_definitions.py @0.40.0](https://github.com/OpenHands/OpenHands/blob/0.40.0/openhands/resolver/interfaces/issue_definitions.py)).
  - Removed from the main repo in April 2026 ("Removed the V0 resolver (#14062)", commit `cc100c0d1`).

### Sweep (retired)

- **Ready signal.** An issue titled `Sweep: …`. "Sweep only considers issues with the 'Sweep:' title on creation and not on update". For existing issues, add the `Sweep` label ([README @2024-05](https://github.com/sweepai/sweep/blob/8d32891fd/README.md)).
- **Feedback.** "Addresses developer replies & comments on its PRs" ([README](https://github.com/sweepai/sweep/blob/8d32891fd/README.md)). The CI loop in `on_failing_github_actions.py` polls check runs once a minute and gives up after 120 minutes of polling. It fetches failing logs and summarises them with an LLM before fixing, and stops after `GHA_MAX_EDIT_ATTEMPTS = 10` ([source](https://github.com/sweepai/sweep/blob/main/sweepai/handlers/on_failing_github_actions.py)).
- **Per-repo customisation.** `sweep.yaml`: `gha_enabled`, `branch`, `blocked_dirs`, `draft`, `description`, `rules`. The docs note that draft PRs mean "GitHub Actions will not be triggered" ([config docs](https://github.com/sweepai/sweep/blob/main/docs/pages/usage/config.mdx)).
- **Status.** The README now points to a JetBrains assistant. The issue-to-PR product is gone ([README](https://github.com/sweepai/sweep/blob/main/README.md)).

### SWE-agent / mini-swe-agent

These are research harnesses, not daemons. `sweagent run --problem_statement.github_url=<issue> --agent.model.per_instance_cost_limit=2.00` reads an issue and saves a patch locally ([hello world](https://swe-agent.com/latest/usage/hello_world/)). There's no Tracker write-back and no PR feedback loop. The README says mini-swe-agent "has superseded SWE-agent" ([README](https://github.com/SWE-agent/SWE-agent/blob/main/README.md)). The lesson: a hard cost cap per instance, and the agent loop itself is commoditising.

### Aider-based bots

Aider itself is a local CLI. `--message` runs one instruction and exits, `--yes` auto-confirms, and auto-commits are on by default. The Python scripting API "is not officially supported" ([scripting](https://aider.chat/docs/scripting.html)). `--test-cmd`/`--auto-test` and `--lint-cmd`: "Aider will try and fix any errors if the command returns a non-zero exit code" ([lint/test](https://aider.chat/docs/usage/lint-test.html)). Source caps this at `max_reflections = 3` ([base_coder.py](https://github.com/Aider-AI/aider/blob/main/aider/coders/base_coder.py)). No first-party issue-to-PR bot was found. Community bots wrap this CLI.

---

## Implications for sous-chef (inputs to the map)

These are observations for later design work on the map, not decisions:

- **Who writes Tracker state: sous-chef or the Coding Agent?** Symphony delegates writes to the agent via host-side tools so the orchestrator stays provider-agnostic. Every hosted product writes status itself. For a single developer with two Trackers and two Coding Agents, sous-chef writing state deterministically is likely simpler to test.
- **What sous-chef owns that Symphony leaves to the prompt.** Detecting new review comments and CI failures, and deciding when "approved and green" is reached. Symphony puts this in the prompt. sous-chef can make it deterministic: poll PR reviews, review threads and check runs, and start an iteration only on new actionable input.
- **Ready as an edge, plus dedupe.** Devin's edge detection and OpenHands' persistent dedupe both avoid re-dispatching a Work Item that already matches. Symphony gets the same effect from claims plus Tracker state.
- **A default workflow plus a per-repo override file.** This matches Symphony's `WORKFLOW.md` (built-in default prompt when absent; strict template; hot reload) and Sweep's `sweep.yaml`.
- **Local and headless brings some issues the hosted tools don't have.** sous-chef pushes with the user's own git credentials, so CI does trigger (unlike `GITHUB_TOKEN`). But there's no VM or firewall, so the worktree path invariant, credential scrubbing from the Coding Agent's env, and trust filtering (write-access authors only, bot allowlist) matter more.

## Sources

- OpenAI Symphony: [README](https://github.com/openai/symphony/blob/main/README.md), [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md), [elixir/WORKFLOW.md](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md)
- GitHub Copilot: [about cloud agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent), [kick off a task](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/kick-off-a-task), [risks and mitigations](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/risks-and-mitigations), [get the best results](https://docs.github.com/en/copilot/tutorials/cloud-agent/get-the-best-results), [Agent Merge](https://docs.github.com/en/copilot/how-tos/copilot-on-github/use-copilot-agents/get-pull-requests-ready-to-merge), [Jira changelog](https://github.blog/changelog/2026-03-05-github-copilot-coding-agent-for-jira-is-now-in-public-preview/)
- claude-code-action: [usage.md](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md), [security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [capabilities-and-limitations.md](https://github.com/anthropics/claude-code-action/blob/main/docs/capabilities-and-limitations.md), [faq.md](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)
- Cursor: [cloud agents](https://cursor.com/docs/cloud-agent), [security overview](https://cursor.com/docs/cloud-agent/security.md), [best practices](https://cursor.com/docs/cloud-agent/best-practices.md), [Linear](https://cursor.com/docs/integrations/linear), [Bugbot](https://cursor.com/docs/bugbot.md), [CLI headless](https://cursor.com/docs/cli/headless)
- Devin: [GitHub](https://docs.devin.ai/integrations/gh.md), [Linear](https://docs.devin.ai/integrations/linear.md), [bot comment settings](https://docs.devin.ai/product-guides/bot-comment-settings.md)
- OpenHands: [issue to PR](https://docs.openhands.dev/openhands/usage/agent-canvas/prebuilt/issue-to-pr.md), [software factory](https://docs.openhands.dev/openhands/usage/use-cases/software-factory.md), [resolver README @0.40.0](https://github.com/OpenHands/OpenHands/blob/0.40.0/openhands/resolver/README.md), [issue_definitions.py @0.40.0](https://github.com/OpenHands/OpenHands/blob/0.40.0/openhands/resolver/interfaces/issue_definitions.py)
- Sweep: [README @2024-05](https://github.com/sweepai/sweep/blob/8d32891fd/README.md), [current README](https://github.com/sweepai/sweep/blob/main/README.md), [config docs](https://github.com/sweepai/sweep/blob/main/docs/pages/usage/config.mdx), [on_failing_github_actions.py](https://github.com/sweepai/sweep/blob/main/sweepai/handlers/on_failing_github_actions.py)
- SWE-agent: [README](https://github.com/SWE-agent/SWE-agent/blob/main/README.md), [hello world](https://swe-agent.com/latest/usage/hello_world/)
- Aider: [scripting](https://aider.chat/docs/scripting.html), [lint/test](https://aider.chat/docs/usage/lint-test.html), [base_coder.py](https://github.com/Aider-AI/aider/blob/main/aider/coders/base_coder.py)
