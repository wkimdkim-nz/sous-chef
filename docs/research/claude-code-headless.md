# Driving Claude Code headlessly

Research for [#2](https://github.com/wkimdkim-nz/sous-chef/issues/2), part of [#1](https://github.com/wkimdkim-nz/sous-chef/issues/1).

**Question:** How can sous-chef drive Claude Code (as a Coding Agent) headlessly from a Node/TypeScript process, and what does that surface support?

**As of:** 2026-11-04. Versions checked: Claude Code CLI 2.1.274 installed locally (`claude --version`), npm `@anthropic-ai/claude-code` latest 2.1.318, npm `@anthropic-ai/claude-agent-sdk` latest 0.3.318 (published 2026-11-04). Both surfaces ship almost daily and the docs gate many behaviours on specific CLI versions (noted inline as "v2.1.x+"). Pin versions and re-check before relying on a gated behaviour.

**Terminology:** a Claude Code *session* is Claude Code's own persisted conversation (a `.jsonl` transcript keyed by a session ID). A sous-chef **Run** may span several Claude Code sessions, or resume one across feedback iterations. This doc keeps the two apart.

---

## TL;DR

1. **Use the TypeScript Agent SDK (`@anthropic-ai/claude-agent-sdk`), not `claude -p` via `child_process`.** Both run the same agent loop: the SDK spawns a bundled native Claude Code binary. The SDK gives typed messages, in-process hooks and `canUseTool` callbacks, `AbortController`, and session helpers. `claude -p --output-format stream-json` is the documented fallback for non-JS callers. [S1][S2][S3]
2. **Worktree:** set `cwd` to the Run's worktree. Project config (`CLAUDE.md`, `.claude/rules`, `.claude/skills`, `.claude/settings.json` hooks, `.mcp.json`) loads from `cwd` by default. Pass `settingSources: []` to load none of it. [S4]
3. **Resume works:** capture `session_id` from the result and pass `resume: id` on the next feedback iteration, which gets the full prior conversation. Transcripts live on local disk under `~/.claude/projects/<encoded-cwd>/` and are swept after 30 days by default. [S5][S11]
4. **Permissions for unattended runs:** a headless run starts in `default` (Manual) mode, which denies anything needing approval unless something approves it (v2.1.302+). Combine an explicit `allowedTools` list, `disallowedTools`, `permissionMode` (`dontAsk` or `acceptEdits`), and `permissionPrompts: 'none'`. Add a `PreToolUse` hook for checks that must cover every call. [S6][S2]
5. **Blocking pushes to the default branch:** permission rules match command text only. The docs give `git -C . push` as an example that slips past `Bash(git push *)`. Layer: (a) a `PreToolUse` hook that inspects every Bash command, (b) sous-chef doing the push itself, (c) server-side branch protection, which is the only real guarantee. [S7][S12]
6. **Limits:** `maxTurns`, `maxBudgetUsd` (a client-side estimate that can overshoot), Bash per-command timeouts (2 min default, 10 min max). No wall-clock limit is built in, so sous-chef enforces one with `AbortController` or `close()`. [S8][S13]
7. **Accounting:** every result carries `total_cost_usd`, `usage`, `modelUsage`, `num_turns`, `duration_ms`. `modelUsage` covers subagents and `usage` does not. A result from a resumed session reports the session's cumulative total (v2.1.277+). [S9]
8. **Auth:** the docs steer Agent SDK use to API keys. The subscription long-lived token (`claude setup-token` → `CLAUDE_CODE_OAUTH_TOKEN`) exists for "CI pipelines and scripts". For a personal, single-user tool on your own machine and plan, that falls under "ordinary, individual usage", but the boundary is policy, not code. **Default to `ANTHROPIC_API_KEY`.** [S10][S14][S1]
9. **Outcome detection:** don't trust `subtype` alone. A local smoke test (CLI 2.1.274) got an auth failure with `subtype: "success"` and `is_error: true`. Classify on `is_error`, `subtype`, `terminal_reason`, `stop_reason`, `permission_denials` and, for the "what happened" payload, `structured_output` from a JSON schema. [S2][S3][local test]
10. **"Needs human":** two mechanisms. (a) A `PreToolUse` hook returns `defer` on `AskUserQuestion`: the run ends with `stop_reason: "tool_deferred"` plus a `deferred_tool_use` payload, and resumes later. (b) Simpler: require a structured-output field such as `status: "needs_human"`. [S11][S3]

---

## 1. `claude -p` vs the Agent SDK

| | `claude -p` (CLI print mode) | Agent SDK (TypeScript) |
|---|---|---|
| What it is | Non-interactive CLI. The docs call it "the Agent SDK via the CLI". [S1] | npm library that "runs the Claude Code binary". The native binary ships as a per-platform optional dependency (e.g. `@anthropic-ai/claude-agent-sdk-darwin-arm64`), so no separate install is needed. Override with `pathToClaudeCodeExecutable`. [S2][S3] |
| Node requirement | n/a | `engines.node >= 18.14.0`. Peer deps: `zod ^4`, `@anthropic-ai/sdk >=0.93`, `@modelcontextprotocol/sdk ^1.29` (`npm view`, 0.3.318). |
| Output | `--output-format text\|json\|stream-json`. `stream-json` emits NDJSON events and ends with a `result` line. `--include-partial-messages` adds token deltas. [S1] | Async iterator of typed `SDKMessage`s. Same event set: `system/init`, `assistant`, `user`, `result`, `stream_event`, `api_retry`, `compact_boundary`, task and hook events, ... [S3] |
| Permission callbacks | Only via an MCP `--permission-prompt-tool`, or settings-file hooks (shell commands). [S1][S15] | In-process `canUseTool` callback and in-process hook callbacks (`PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, ...). [S6][S4] |
| Control | Signals only. SIGTERM exits 143 and leaves the turn unfinished with no result. SIGINT ends the turn. [S1] | `AbortController`, `interrupt()` (streaming input only), `close()`, and `setPermissionMode()` / `setModel()` mid-session (streaming input only). [S3] |
| Exit status | 0 on success, non-zero on failure. In-run failures (e.g. auth) are printed as the result on stdout. [S1] | Single-shot `query()` yields the error result, then throws. Wrap it in try/catch. [S8] |

**Trade-off for a long-running supervisor:** the SDK wins. It removes NDJSON parsing and signal plumbing, and it lets sous-chef enforce policy (push guard, Run timeouts, "needs human") in-process. The cost: the SDK moves fast, with near-daily releases and breaking changes. For example, the V2 `createSession()` API was removed in 0.3.142 [S5]. So pin the SDK version, which also pins the bundled CLI. Each Run is still a child process, so a crash in Claude Code does not take the supervisor down.

The docs recommend `--bare` for "scripted and SDK calls" and say it "will become the default for `-p` in a future release" [S1]. Bare mode skips hooks, skills, plugins, MCP, auto memory and CLAUDE.md, and it **ignores OAuth/keychain** (API key or `apiKeyHelper` only). This is version-sensitive: if the default flips, CLAUDE.md loading silently stops unless sous-chef opts in explicitly.

## 2. Working directory, worktree and project config

- `cwd` option (default `process.cwd()`) sets where project inputs load from. `additionalDirectories` maps to `--add-dir`. [S3][S4]
- When `settingSources` is omitted, it behaves like `["user","project","local"]`, so the SDK loads exactly what the CLI loads. [S4]
  - `"project"`: `<cwd>/.claude/settings.json` and its hooks; `CLAUDE.md` and `.claude/rules/*.md` in `<cwd>` and every parent; skills, commands and subagents from `<cwd>` up to the repo root.
  - `"user"`: `~/.claude/settings.json`, `~/.claude/CLAUDE.md`, user skills and agents.
  - `"local"`: `CLAUDE.local.md`, `.claude/settings.local.json`.
  - These load **regardless** of `settingSources`: managed policy, `~/.claude.json`, auto memory at `~/.claude/projects/<project>/memory/` (disable with `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`), and claude.ai MCP connectors when logged in with claude.ai.
- Skills: discovered via `settingSources`. The `skills` option takes `"all"`, a list of names, or `[]`. There is no programmatic skill registration; skills must be `SKILL.md` files on disk. [S4]
- Plugins: `plugins: [{ type: 'local', path }]` (CLI: `--plugin-dir`). MCP: the `mcpServers` option. Set `strictMcpConfig: true` to ignore `.mcp.json`. [S3]
- **Trust caveat for `-p` and SDK runs:** no workspace-trust dialog is shown. Project hooks, the `env` block and `apiKeyHelper` **are run**, and `.mcp.json` servers connect without asking. But **project `permissions.allow` rules in `.claude/settings.json` are *not* used** until the folder is trusted (a `this workspace has not been trusted` warning goes to stderr). sous-chef should therefore pass its allow-list explicitly rather than rely on the target repo's settings. Trust can be set by hand via `projects["<path>"].hasTrustDialogAccepted` in `~/.claude.json`. [S7]
- Startup health: the `system/init` message reports `cwd`, `tools`, `mcp_servers[].status`, `plugins`, `plugin_errors`, `mcp_server_errors`, `skills`, `permissionMode`, and `claude_code_version`. Gate a Run on these. [S1][S3]
- If the worktree directory is deleted mid-session, the session keeps running but shell commands fail. [S1]
- The CLI also has `-w/--worktree` to create its own worktree under `<repo>/.claude/worktrees/<name>`. sous-chef should manage worktrees itself and just set `cwd`. [S15]

## 3. Session resume and continuation

- Every result (success or error) carries `session_id`, and the `system/init` message carries it too. Pass `resume: sessionId` to continue with full context. `forkSession: true` branches into a new ID. `continue: true` picks the most recent session in the directory. Pre-assigning an ID with `sessionId: <uuid>` is also supported (CLI `--session-id`). [S5][S3]
- Storage: `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`, where every non-alphanumeric character of the absolute cwd becomes `-`, or under `$CLAUDE_CONFIG_DIR/projects/`. Lookup searches beyond the current project directory (v2.1.223+), so resuming from a different cwd works, but only **on the same machine**. A `sessionStore` adapter mirrors transcripts elsewhere. [S5]
- Retention: `cleanupPeriodDays` deletes session files after 30 days by default. [S11]
- Sessions persist the **conversation, not the filesystem**. The worktree must still be in the state the agent left it in. [S5]
- When resuming with `-p`, the stored permission mode is **not** restored. Pass `--permission-mode` again. [S11] The SDK passes options per call anyway.
- `persistSession: false` (TS) or `--no-session-persistence` makes a session non-resumable. [S3][S15]
- **Implication for sous-chef:** a feedback iteration (review comments, CI failure) can `resume` the Run's previous session ID, as long as sous-chef stores it per Run and resumes within 30 days on the same host. The docs also suggest not relying on resume and passing the needed state into a fresh prompt, which is "often more robust" [S5]. A reasonable design keeps both: resume by default, fall back to a fresh session seeded with a summary.

## 4. Permissions, sandboxing, forbidding pushes

**Evaluation order** (SDK): hooks → deny rules → ask rules → permission mode → allow rules → `canUseTool`. A deny rule wins even in `bypassPermissions` mode and even when a hook returned allow. [S6]

**Modes** (`permissionMode` / `--permission-mode`): `default`, `dontAsk`, `acceptEdits`, `bypassPermissions` (TS also requires `allowDangerouslySkipPermissions: true` and refuses to run as root), `plan`, `auto` (a classifier reviews actions; model and provider dependent). The local CLI help also lists `manual`. When nothing sets a mode, headless runs start in `default`/Manual (v2.1.302+), which denies unapproved calls when no callback is present. [S6][S8][local `--help`]

**Rules:**
- `allowedTools` auto-approves; it does **not** restrict which tools exist. With `bypassPermissions` it constrains nothing. [S6]
- `disallowedTools: ["Bash"]` removes the tool. `["Bash(rm *)"]` denies matching calls in every mode. `tools: [...]` sets the available built-ins. [S6][S3]
- `permissionPrompts: 'none'` (CLI `--permission-prompts none`, v2.1.259+) denies anything that would prompt, tells Claude not to retry, and removes `AskUserQuestion`. [S1][S6]
- A locked-down recipe from the docs: `allowedTools` plus `permissionMode: "dontAsk"`. [S6]

**Bash rule limits:** rules match command text after compound splitting and wrapper stripping. A rule is "not a security boundary around the program". Per the docs' table, `Bash(git push *)` stops `git push origin main` but not `git -C . push origin main`, `git -c push.default=current push origin main`, or `git 'push' origin main`. The docs point to the sandbox or a `PreToolUse` hook for real enforcement. [S7]

**Hooks:** a `PreToolUse` hook runs before every tool call. It can return `permissionDecision: allow|deny|ask|defer` with `updatedInput`, and in shell hooks exit code 2 blocks. In the SDK, hooks are TS callbacks. [S4][S11]

**Sandbox** (`sandbox` option, or `sandbox.*` in settings): off by default. It is OS-level (macOS Seatbelt, Linux bubblewrap) and wraps Bash/PowerShell/Monitor commands and their children. Writes are limited to cwd plus allowed dirs, and `.git/hooks` and `.claude` settings stay write-denied. Network goes through a proxy with an empty `allowedDomains` by default. `failIfUnavailable: true` refuses to start unsandboxed, and `allowUnsandboxedCommands: false` removes the `dangerouslyDisableSandbox` escape hatch. Caveats on macOS: `git` over SSH fails inside the sandbox (use HTTPS or exclude git push/fetch), and Go CLIs like `gh` may fail TLS. The docs suggest adding them to `excludedCommands`, which runs them *unsandboxed*. [S12]

**Forbidding pushes to the default branch.** Recommendation, in layers:
1. **sous-chef owns `git push` and PR creation.** Deny the agent's push outright: `disallowedTools: ["Bash(git push *)"]` plus a `PreToolUse` hook that rejects any Bash command whose parsed argv contains `push` after `git` (catches `git -C`, `-c`, quoting), and ideally also `gh pr merge`, `gh api` writes and so on. The agent commits on the Run branch; sous-chef pushes that branch itself, so the agent never needs remote write access.
2. If the agent must push, the hook allows only `git push <remote> <run-branch>` and denies any refspec naming the default branch, `HEAD:` forms, `--all`, `--mirror`.
3. **Server-side:** branch protection or a ruleset on the default branch (require PRs, block direct pushes). This is the only control that doesn't depend on parsing commands. [S16] (Rulesets' plan availability varies; check it for the repo.)
4. Optionally, the sandbox's network allowlist omits github.com so an unsandboxed-retry push can't reach the remote.

## 5. Limits, timeouts, accounting, auth

**Limits** [S8][S15][S13]
- `maxTurns` (CLI `--max-turns`, print-only, hidden from `--help` in 2.1.274 but documented). The final text-only response counts as a turn. Exceeding it gives `error_max_turns`.
- `maxBudgetUsd` (CLI `--max-budget-usd`): checked after each response, so spend can exceed it by one response plus running subagents. It includes subagents. Exceeding it gives `error_max_budget_usd`.
- Bash per-command timeout: `BASH_DEFAULT_TIMEOUT_MS` (2 min) and `BASH_MAX_TIMEOUT_MS` (10 min). A foreground command that times out is moved to the background (30 min limit, 2 h max) unless background tasks are disabled.
- At exit a `-p` run waits up to 10 min idle for background work (`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`). [S1]
- There is **no overall wall-clock timeout** option. sous-chef enforces one via `abortController.abort()` or `close()`, and should end the turn gracefully first (SIGINT/`interrupt()`) so a result is recorded. [S1][S3]
- `effort` (`low` … `max`) trades depth for cost. [S8]

**Accounting** [S9]
- Result fields: `total_cost_usd`, `usage` (main loop only), `modelUsage` (per model, includes subagents, with `costUSD` and a `costBasis` of `list`/`managed`/`unknown`), `num_turns`, `duration_ms`, `duration_api_ms`.
- Costs are **client-side estimates** from a bundled price table. They can drift from the bill, so don't use them for financial decisions.
- Resumed sessions report cumulative totals (v2.1.277+). Read the latest result; don't sum. A crash result may carry zeroed costs.
- Per-step `output_tokens` on assistant messages is a placeholder; take output tokens from the result.

**Auth** [S10][S14][S1]
- Precedence: cloud provider env (`CLAUDE_CODE_USE_BEDROCK`/`_VERTEX`/`_FOUNDRY`) → `ANTHROPIC_AUTH_TOKEN` → `ANTHROPIC_API_KEY` (always used in `-p` when present) → `apiKeyHelper` → `CLAUDE_CODE_OAUTH_TOKEN` → Anthropic profiles → `/login` subscription OAuth.
- `claude setup-token` mints a one-year subscription OAuth token (Pro/Max/Ultra/Team/Enterprise) "for CI pipelines and scripts". It can only make model requests (no claude.ai connectors or Remote Control) and is **not read in `--bare`**.
- Policy: the SDK overview says third-party developers may not "offer claude.ai login or rate limits" for their products, and the legal page says developers "building products or services ... including those using the Agent SDK" should use API keys. The same page also says advertised plan limits "assume ordinary, individual usage of Claude Code and the Agent SDK", and it doesn't prevent "an end user from signing in to the unmodified Claude Code binary with their own Claude subscription".
- **Implications:** for a personal tool that runs on Derek's machine with Derek's own plan, the subscription token is plausibly within "ordinary, individual usage", but heavy unattended polling could stretch that and runs into plan rate limits. An API key gives clean per-token billing, `--bare` compatibility, and no policy ambiguity. If sous-chef is ever distributed, it must not intermediate users' claude.ai credentials. Each user brings their own key or login. Note also that the local `/login` OAuth expired during this research (`Failed to authenticate: OAuth session expired`), which is exactly the failure mode an unattended supervisor has to detect.

## 6. Output and outcome detection

- `SDKResultMessage` subtypes: `success` | `error_max_turns` | `error_during_execution` | `error_max_budget_usd` | `error_max_structured_output_retries`. Only `success` carries `result` and `structured_output`; error results carry `errors: string[]`. Every result carries `is_error`, `num_turns`, `session_id`, `stop_reason`, cost fields, `permission_denials[]` (described as authoritative), an optional `terminal_reason` (`completed`, `max_turns`, `tool_deferred`, `aborted_*`, `hook_stopped`, `budget_exhausted`, `api_error`, `prompt_too_long`, ...), `api_error_status`, and `deferred_tool_use`. [S3]
- **Observed (CLI 2.1.274, `claude -p --output-format json`):** an expired OAuth login produced `"subtype":"success"`, `"is_error":true`, `"terminal_reason":"api_error"`, `"result":"Failed to authenticate: ..."`. **Treat `is_error` as the primary success flag.**
- `stop_reason: "refusal"` means the model declined. [S8]
- Iterate the stream to completion. Trailing events such as `prompt_suggestion` can follow the result. [S8]
- **Structured output:** `outputFormat: { type: 'json_schema', schema }` (CLI `--json-schema`) puts validated JSON in `structured_output`. If none is produced, the field is absent or the subtype is `error_max_structured_output_retries`, so check for it. `format` keywords are not enforced. [S1][S3]
- **Streaming:** `includePartialMessages` gives `stream_event` deltas. `system/api_retry` events report retries with an error category (`rate_limit`, `authentication_failed`, `billing_error`, ...), which is useful for telling infra failures apart from task failures. Subagent messages carry `parent_tool_use_id`. [S1]
- **"Needs human":**
  - Built-in: in a `-p`/SDK run with a permission host, `AskUserQuestion` reaches `canUseTool`. A `PreToolUse` hook can return `defer`, which ends the turn with `stop_reason: "tool_deferred"` and `deferred_tool_use {id,name,input}`. Resume later without a prompt and the hook can then `allow` with the answer in `updatedInput`. This only works when Claude makes a single tool call in that turn. A `canUseTool` callback can also simply stay pending. [S11][S17]
  - Pragmatic: since a Run has no human in the loop, use `permissionPrompts: 'none'` (which removes `AskUserQuestion`) and require a structured-output `status` field (`done` / `needs_human` / `failed`, plus `reason` and a PR summary). Classify a Run from `is_error`, `subtype`, `terminal_reason`, `permission_denials` and `structured_output.status`.

---

## Sources

- **[S1]** Run Claude Code programmatically (headless): https://code.claude.com/docs/en/headless
- **[S2]** Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview
- **[S3]** Agent SDK TypeScript reference (`Options`, `Query`, `SDKResultMessage`, `SDKSystemMessage`, `SDKMessage`): https://code.claude.com/docs/en/agent-sdk/typescript, plus `npm view @anthropic-ai/claude-agent-sdk` (0.3.318: engines, optionalDependencies, peerDependencies)
- **[S4]** Use Claude Code features in the SDK (`settingSources`, CLAUDE.md, skills, hooks): https://code.claude.com/docs/en/agent-sdk/claude-code-features
- **[S5]** Work with sessions: https://code.claude.com/docs/en/agent-sdk/sessions
- **[S6]** SDK permissions: https://code.claude.com/docs/en/agent-sdk/permissions
- **[S7]** Configure permissions (Bash rule limits, hooks precedence, "What runs before you trust a folder"): https://code.claude.com/docs/en/permissions
- **[S8]** How the agent loop works (turns, budget, result subtypes, effort, modes): https://code.claude.com/docs/en/agent-sdk/agent-loop
- **[S9]** Track cost and usage: https://code.claude.com/docs/en/agent-sdk/cost-tracking
- **[S10]** Authentication (precedence, `setup-token`): https://code.claude.com/docs/en/authentication
- **[S11]** Hooks reference ("Defer a tool call for later", PreToolUse decisions): https://code.claude.com/docs/en/hooks
- **[S12]** Sandboxing: https://code.claude.com/docs/en/sandboxing
- **[S13]** Tools reference (Bash timeouts, background limits): https://code.claude.com/docs/en/tools-reference
- **[S14]** Legal and compliance (authentication and credential use): https://code.claude.com/docs/en/legal-and-compliance
- **[S15]** CLI reference: https://code.claude.com/docs/en/cli-reference, plus local `claude --help` (2.1.274)
- **[S16]** GitHub, About rulesets: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets
- **[S17]** Handle approvals and user input (`canUseTool`, `AskUserQuestion`): https://code.claude.com/docs/en/agent-sdk/user-input
- **[local test]** `claude -p "Reply with exactly: OK" --output-format json --max-turns 1 --no-session-persistence --tools ""`, CLI 2.1.274, 2026-11-04. Result was an auth failure; the JSON shape is quoted in §6.
