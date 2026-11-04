# Driving Cursor headlessly

Research for [#3](https://github.com/wkimdkim-nz/sous-chef/issues/3) (part of map #1). Researched 2026-11-04.

**Question:** How can sous-chef drive **Cursor** headlessly from a Node/TypeScript process, and what does that surface support?

## Short answer

Cursor has two headless surfaces, both usable from Node:

1. **The Cursor CLI** (`agent -p ...`): a subprocess with `text` / `json` / `stream-json` (NDJSON) output. The shape is close to `claude -p`.
2. **`@cursor/sdk`** (npm, `latest` = 1.3.1, Node >= 22.13): an in-process TypeScript API. It has `Agent.create` / `Agent.resume`, `run.stream()` / `run.wait()` / `run.cancel()`, a typed `RunResult.status` (`finished | error | cancelled`), and cumulative token `usage`. Only the agent loop runs locally. Inference still runs on Cursor's hosted models.

Both authenticate unattended with a user API key (`CURSOR_API_KEY`). Both run in a directory you choose and load `.cursor/rules`, `AGENTS.md` and MCP config. Both can resume a session by ID.

The parity gaps against Claude Code that matter for sous-chef:

- The Cursor CLI has **no turn or budget caps** and **no cost in its output**.
- Cursor has **no documented exit codes** and **no failure subtypes**. A failure shows up as a non-zero exit plus stderr, with no `result` event.
- Cursor has **no structured-output / JSON-schema flag**.
- Cursor has **no "permission denials" report**.
- The permission model differs. Cursor uses `Shell(...)` / `Read(...)` / `Write(...)` allow/deny lists plus a Run Mode (`--force` = Run Everything, `--auto-review`, allowlist). Claude Code uses `--permission-mode` plus `--allowedTools` / `--disallowedTools`.

> Version note: the CLI auto-updates by default and ships often. The latest CLI changelog entry is **v2026.10.22**. Everything below is as documented on 2026-11-04. The local machine does **not** have the CLI installed (`agent`, `cursor-agent` and `cursor` are all missing from PATH), so `--help` was not checked. All CLI facts come from the official docs and changelog.

---

## 1. Non-interactive invocation, auth, model selection

**Binary.** The current docs name the binary `agent`. The installer is `curl https://cursor.com/install -fsS | bash`, which puts it on PATH via `~/.local/bin`. `agent update` updates it, and *"the CLI tries to update itself by default"*. The installation page does not document a way to pin a version. [installation] The product was formerly `cursor-agent`. The current docs never mention that name, so sous-chef should resolve `agent` and fall back to `cursor-agent`.

**Headless mode.** `-p, --print` runs non-interactively and "has access to all tools, including write and shell". `--output-format text|json|stream-json` requires `--print` and defaults to `text`. `--stream-partial-output` adds text deltas to `stream-json`. [parameters] [headless]

```bash
agent -p --force --trust --workspace "$WORKTREE" \
  --model "$MODEL" --output-format stream-json "<prompt>"
```

- Without `--force`, the headless docs say the agent only *proposes* edits and "won't modify files". Pass `--force` (alias `--yolo`) to let it apply edits and run commands "without confirmation unless explicitly denied". [headless] [parameters]
  - This conflicts with the "Using Agent in CLI" page, which says print mode "has full write access". The `--force` example is the one to trust. Treat `--force` as required for a Coding Agent Run.
- **Workspace trust.** Since January 2026, non-interactive runs in untrusted workspaces *fail* unless `--trust` or `--force` is passed. A fresh git worktree per Run is a new, untrusted path, so always pass `--trust`. `--trust` is "headless only", though v2026.07.20 extended it to interactive sessions. [changelog Jan 2026, v2026.07.20] [parameters]
- **No stdin hang.** Since February 2026, `-p` runs no longer hang when spawned with an open stdin pipe. That is relevant for `child_process.spawn`. [changelog Feb 2026]

**Auth for unattended use.**

- Create a **user API key** at `cursor.com/dashboard/api`.
- Pass it as the `CURSOR_API_KEY` env var (recommended) or `--api-key <key>`. [authentication] [parameters]
- `agent status` / `whoami` reports auth state, and `--format json` works for status in headless mode. [parameters] [changelog Apr 2026]
- The `system/init` stream event reports `apiKeySource` (`env|flag|login`). [output-format]
- On machines without a keychain, `AGENT_CLI_CREDENTIAL_STORE=file` stores credentials in an owner-only file. [changelog v2026.06.29]
- **Team admins can disable headless mode org-wide.** That only matters on a Teams plan. [changelog v2026.06.09]

**Model selection.**

- Use `--model <model>`. `--list-models` and `agent models` list the models the account can use.
- Team model restrictions apply: a blocked model makes the CLI exit with a message. [changelog v2026.08.26]
- Max-mode-only models get Max Mode sent automatically in headless runs. [changelog v2026.08.11]
- Fast and high-effort variants keep their parameters in `-p`. [changelog May 14 2026]
- The SDK requires `model: { id: "composer-2.5" }`-style config before the first `send()`. [sdk]

## 2. Working directory / worktree; project config

- **Working directory.** `--workspace <path>` sets the workspace. Otherwise the CLI uses the cwd. [parameters] [using] The SDK uses `local.cwd`. [sdk]
- **Built-in worktrees (not recommended).** `-w, --worktree [name]` creates a worktree under `~/.cursor/worktrees/<reponame>/<name>`. `--worktree-base <branch>` sets its base, and `.cursor/worktrees.json` setup scripts run unless you pass `--skip-worktree-setup`. Cleanup follows Cursor's own retention rules. [parameters] [using] sous-chef should create and own its worktree (one per Run) and pass `--workspace`. That gives the same lifecycle for every Coding Agent, and Cursor's retention rules can't delete a worktree mid-Run.
- **Rules.** The CLI loads `.cursor/rules` and also reads `AGENTS.md` and `CLAUDE.md` at the project root. [using]
- **MCP.** The CLI reads `mcp.json` (`.cursor/mcp.json` and `~/.cursor/mcp.json`). [using] [parameters]
  - `--approve-mcps` auto-approves all MCP servers, and `agent mcp enable|disable|list` manages them. [parameters]
  - `--yolo` also auto-approves trust and MCP prompts. [changelog Feb 2026]
  - Headless `-p` waits for slow stdio MCP servers to start. [changelog May 14 2026]
  - Local OAuth MCP servers can't prompt for sign-in headlessly. [sdk]
- **SDK config loading.** The SDK loads nothing from disk unless `local.settingSources` includes `"project"` (`.cursor/`), `"user"` (`~/.cursor/`) or `"plugins"`. [sdk]
- **Config that applies only globally.** The project file `<project>/.cursor/cli.json` can set **only permissions**. Everything else in the CLI config is global: `~/.cursor/cli-config.json`, or a different directory via `CURSOR_CONFIG_DIR` / `XDG_CONFIG_HOME`. [configuration] Running each Run with its own `CURSOR_CONFIG_DIR` is one way to isolate settings per Run. That is inferred, not documented as a pattern.
- **Attribution.** By default, agent commits get a "Made with Cursor" trailer and agent PRs get a footer. The `attribution.attributeCommitsToAgent` / `attributePRsToAgent` settings control them. [configuration]

## 3. Session resume / continuation

- **CLI.**
  - `--resume [chatId]` resumes a specific chat. `--continue` (= `--resume=-1`) and `agent resume` resume the latest one.
  - `agent create-chat` creates an empty chat and returns its ID up front, which works like Claude Code's `--session-id`.
  - The `json` and `stream-json` outputs carry `session_id`. [parameters] [output-format]
  - Since v2026.07.06, resume finds chats from all workspaces and loads the full conversation. [changelog]
  - Auto-run settings are recomputed from config and team policy when a conversation resumes. [changelog May 14 2026]
- **Recommended flow.** Store `session_id` (or the ID from `create-chat`) on the Run. For each feedback iteration (review comment, CI failure), run `agent -p --resume <id> --workspace <same worktree> ...`.
- **SDK.**
  - `Agent.resume(agentId)` works from a new process. Local IDs look like `agent-<uuid>`, and `bc-` prefixes mean cloud agents.
  - State is stored in SQLite by default, or in `JsonlLocalAgentStore`.
  - Several things are **not** persisted and must be passed again on every resume: `model`, inline `mcpServers`, `systemPrompt`, and `teamId`. [sdk] [sdk release notes 1.0.40]
- **No session fork in the CLI.** The CLI has no equivalent of Claude Code's `--fork-session`. Interactive `/fork` exists (v2026.06.09), but the CLI exposes no headless fork flag.

## 4. Permissions, auto-approval, sandboxing, protecting the default branch

**Permission lists.** `permissions.allow` / `permissions.deny` live in `~/.cursor/cli-config.json` or `<project>/.cursor/cli.json`. [permissions]

- `Shell(cmd)` matches the command's **first token**, so `Shell(git)` covers all git subcommands.
- `cmd:args` globs match arguments too, for example `Shell(curl:*)`.
- The other rule types are `Read(glob)`, `Write(glob)`, `WebFetch(domain)` and `Mcp(server:tool)`.
- **Deny beats allow.**
- `--force` runs everything "unless explicitly denied", so deny rules still apply under `--force`. [parameters]
- Rules are exact strings in JSON with no comments. [configuration]

**Run Modes / approval.** `approvalMode` is one of `allowlist`, `auto-review` or `unrestricted`. [configuration]

- **Run Everything** is `--force` / `--yolo` (March 2026).
- **Auto-review** (`--auto-review`, v2026.06.22; not yet listed on the parameters page) sends calls that aren't allowlisted or sandboxable to a classifier on Cursor's backend (Gemini 3.5 Flash Lite, falling back to Claude 4.5 Haiku). You steer it with `autoRun.allow_instructions` / `block_instructions` sentences in `permissions.json`. Cursor says explicitly that it is **not a security boundary**. [run-modes] [changelog]
- Team-configured approval modes apply in headless runs. [changelog Mar 2026]
- The SDK has **no interactive approval**: tool calls run unapproved by default. You gate them with hooks, `local.sandboxOptions` or `local.autoReview`. [sdk]

**Sandbox.** Use `--sandbox enabled|disabled` or `agent sandbox enable|disable`. Configure it in `~/.cursor/sandbox.json` and `.cursor/sandbox.json`. The two merge, the project file wins, and team policy can't be weakened locally. [parameters] [run-modes]

- macOS uses Seatbelt. Linux uses Landlock + seccomp (kernel 6.2+) with a Bubblewrap fallback, and the standalone CLI may need the `cursor-sandbox-apparmor` profile.
- Sandboxed commands can read and write the workspace. **Network is blocked by default, except Cursor's default domain list, which includes github.com.**
- `.git/config` and `.git/hooks` are protected paths.
- A command that fails inside the sandbox can be re-run outside it, after classifier review.
- The CLI fails fast if sandboxing is enabled but unavailable. [changelog Feb 2026]

**Hooks.** `.cursor/hooks.json` (project) and `~/.cursor/hooks.json` (user) support `beforeShellExecution` (input includes `command`, `cwd` and `sandbox`; output is `permission: allow|deny|ask`), `preToolUse`, `stop`, `sessionStart`, `sessionEnd` and more. Exit code `2` blocks, and other non-zero codes fail open unless `failClosed` is set. [hooks]

- The hooks page doesn't say which hooks fire in CLI or headless runs.
- The CLI changelog shows hooks *are* supported in the CLI, including **Claude Code `settings.json` hooks**. [changelog Jan 2026]
- Hook payloads carry per-turn token usage and a stable session ID. [changelog Mar 2026]
- In the SDK, a `PreToolUse` deny blocks local tool calls (1.0.40). [sdk release notes]

**Stopping pushes to the default branch.** Cursor has no dedicated control for this. Layer the following, strongest first:

1. **Don't give the Coding Agent push credentials, and have sous-chef do the push itself.** Otherwise apply GitHub branch protection or rulesets on the default branch. This is the only control that doesn't depend on the agent.
2. Add a deny rule. `Shell(git:push*)` would be the `cmd:args` form, but whether it matches the way you'd expect is **unverified**. `Shell(git)` would block all git. Deny rules are prefix and glob matches, so a wrapper script can get around them.
3. Add a `beforeShellExecution` hook that denies `git push` to the default branch (`failClosed: true`). The hooks page uses a git-blocking hook as an example.
4. The sandbox is *not* enough on its own: github.com is in the default network allowlist.

## 5. Limits & accounting

- **No turn or budget caps in the CLI.** There is nothing like `--max-turns` or `--max-budget-usd`, and no run timeout is documented. sous-chef must enforce a wall-clock timeout itself (kill the process tree). The SDK has `run.cancel()`, which stops the run and in-flight tool calls. [parameters] [sdk]
- **Internal timeouts.** Shell commands that time out return the output file's path, size and line count (v2026.09.28). Waiting on a background command no longer sleeps the full timeout (v2026.10.15). Headless runs wait for delegated subagents and background work before exiting (Mar and Aug 2026), so a run can last longer than its last visible assistant message. [changelog]
- **Usage reporting.**
  - CLI: `stream-json` includes **per-turn input, output and cache token totals** and a `request_id` (Feb 2026). [changelog]
  - The output-format page doesn't document the exact field names, and the `json` (single object) format has no documented usage field. **Capture `stream-json` and don't rely on `json` for usage.** Verify the field names against a real run.
  - The CLI does **not** report cost.
  - SDK: `RunResult.usage` / `run.usage` give cumulative `TokenUsage` (`inputTokens`, `outputTokens`, `cacheReadTokens`, `cacheWriteTokens`, `totalTokens`, plus optional `reasoningTokens`), and a `usage` event fires per turn. `agent.getUsage()` (billed cost) works only where Cursor has enabled usage access for the account, and otherwise throws `feature_unavailable`. [sdk]
- **Plan and pricing.** CLI usage draws from the **same weekly usage pool** as the editor and Cloud Agents. Overage on Cursor models is billed per million tokens, and third-party models are billed at API list price from prepaid credit on higher tiers. [pricing] The docs say nothing headless-specific beyond the admin kill switch. Expect long unattended Runs to use up the weekly pool.

## 6. Output and detecting the outcome

**`json`**: one object on success. [output-format]

```json
{"type":"result","subtype":"success","is_error":false,"duration_ms":1234,
 "duration_api_ms":1234,"result":"<assistant text>","session_id":"<uuid>","request_id":"<optional>"}
```

**`stream-json`**: NDJSON events. [output-format]

- `system` (`init`, with `apiKeySource`, `cwd`, `session_id`, `model` and `permissionMode`).
- `user`.
- `assistant`.
- `tool_call` (`started` / `completed`, correlated by `call_id`). Tool payloads are keyed by tool, for example `readToolCall` and `writeToolCall`.
- Terminal `result`.
- Partial-output deduplication: a delta has `timestamp_ms` but no `model_call_id`.
- Thinking events are suppressed.
- New fields may be added, so parsers should ignore unknown fields.

**Success, failure and "needs human".**

- **Success**: exit 0 plus a terminal `result` with `subtype:"success"`, `is_error:false`.
- **Failure**: a non-zero exit and an error on stderr. With `json` there's no well-formed object, and with `stream-json` the stream may end without a `result` event. **The docs give no exit-code values and no error subtypes.** [output-format] Treat "no `result` event" or "non-zero exit" as failure, and keep stderr for diagnosis. Since April 2026, errors are also written to the transcript.
- **SDK**: `RunResult.status` is `finished | error | cancelled`, with `error`. The Python SDK adds `error_code` / `error_message`. [sdk] [sdk release notes 1.0.39]
- **"Needs human"**: no native signal. Headless runs have no approval prompts. Under `--force`, denied calls are simply refused, and nothing reports the refusals the way Claude Code's `permission_denials` does. Timed mode-switch approvals auto-reject when unattended (v2026.08.11). sous-chef needs its own protocol: for example, instruct the agent to finish with a sentinel line or write a status file in the worktree, then parse `result`. This is also the portable approach for Claude Code, which has no native "needs human" result either.
- **No structured output.** There's no `--json-schema`. If you need a machine-readable verdict, have the agent write a JSON file or emit a fenced block, and validate it in sous-chef.

## 7. Parity vs Claude Code (`claude -p`)

| Capability | Cursor CLI (`agent -p`) | `@cursor/sdk` | Claude Code |
|---|---|---|---|
| Output formats | text / json / stream-json | typed events + `RunResult` | text / json / stream-json |
| Session ID up front | `agent create-chat` | `agent.agentId` | `--session-id <uuid>` |
| Resume | `--resume <id>`, `--continue` | `Agent.resume(id)` | `--resume`, `--continue`, `--fork-session` |
| Turn / budget caps | none | none (use `cancel()`) | `--max-turns`, `--max-budget-usd` |
| Cost in output | no (tokens per turn in stream-json) | tokens; billed cost gated | `total_cost_usd`, `modelUsage` |
| Failure taxonomy | non-zero exit + stderr, no `result` | `status: error` + `error` | `subtype` `error_max_turns` / `error_during_execution` / `error_max_budget_usd`..., `terminal_reason`, `errors[]` |
| Permission denials reported | no | no | `permission_denials[]` |
| Structured output | no | no | `--json-schema` -> `structured_output` |
| Permission model | allow/deny `Shell/Read/Write/WebFetch/Mcp` + Run Mode (`--force`, `--auto-review`) + sandbox | hooks / sandboxOptions / autoReview | `--permission-mode` (`default/acceptEdits/plan/auto/dontAsk/bypassPermissions`), `--allowedTools`/`--disallowedTools`, `--permission-prompts none` |
| Workspace trust in headless | must pass `--trust` | n/a | no trust dialog under `-p` |
| Reproducible "bare" run | none (project and global config always load; isolate via `CURSOR_CONFIG_DIR`) | opt-in `settingSources` | `--bare` |
| Reads `AGENTS.md` / `CLAUDE.md` | both | via `settingSources` | `CLAUDE.md` |
| System prompt override | none on CLI | `systemPrompt` (gated per account) | `--append-system-prompt`, `--system-prompt` |
| Mid-run steering | none | `run.steer(text)` | stream-json input |
| Model hosting | Cursor-hosted (Bedrock option via `agent bedrock`) | Cursor-hosted | Anthropic API / Bedrock / Vertex / Foundry |

Claude Code sources: [cc-headless] [cc-cli] [cc-sdk-ts].

**What this means for a common Coding Agent interface:**

- Normalize on: start(worktree, prompt, model) -> sessionId; resume(sessionId, prompt); a stream of {assistant text, tool call, usage}; and a terminal outcome of `success | failed | cancelled | timed_out`.
- sous-chef owns: timeouts, budget (token counting where cost is unavailable), "needs human" detection (a sentinel or status file), and push/branch safety (credentials plus branch protection, never agent self-restraint).
- Per-agent adapters translate a sous-chef permission policy into Cursor `permissions.deny` + hooks + `--force --trust --sandbox enabled`, or into Claude Code `--permission-mode` + `--allowedTools` / `--disallowedTools`.
- Cost is an optional field: Claude Code fills it in, and Cursor usually won't.
- For Cursor, choose CLI subprocess or `@cursor/sdk`. The SDK gives typed status, cancel, usage and steering without parsing NDJSON. It needs Node >= 22.13 and adds a native dependency, and its docs and release notes lag npm (the release-notes page tops out at 1.0.40 while npm `latest` is 1.3.1). The CLI matches the Claude Code subprocess model one-to-one. This is a design decision for the map, not settled here.

## Open / unverified

- Exact `stream-json` usage field names (documented only in the changelog). Verify against a real run.
- Whether `Shell(git:push*)` deny patterns match as expected.
- Which hook events fire in CLI `-p` runs. The changelog implies they do, but the hooks page is silent.
- Exit-code values. Undocumented.
- `--help` output was not checked because the CLI isn't installed locally.

## Sources

- [headless] Cursor docs, Headless CLI: https://cursor.com/docs/cli/headless
- [parameters] Cursor docs, CLI parameters: https://cursor.com/docs/cli/reference/parameters
- [output-format] Cursor docs, Output format: https://cursor.com/docs/cli/reference/output-format
- [permissions] Cursor docs, CLI permissions: https://cursor.com/docs/cli/reference/permissions
- [configuration] Cursor docs, CLI configuration: https://cursor.com/docs/cli/reference/configuration
- [authentication] Cursor docs, CLI authentication: https://cursor.com/docs/cli/reference/authentication
- [installation] Cursor docs, CLI installation: https://cursor.com/docs/cli/installation
- [using] Cursor docs, Using Agent in CLI: https://cursor.com/docs/cli/using
- [changelog] Cursor CLI changelog (latest v2026.10.22): https://cursor.com/docs/cli/changelog
- [run-modes] Cursor docs, Run Modes and Sandboxing: https://cursor.com/docs/agent/run-modes
- [hooks] Cursor docs, Hooks: https://cursor.com/docs/agent/hooks
- [sdk] Cursor TypeScript SDK: https://cursor.com/docs/sdk/typescript ; npm `@cursor/sdk` (latest 1.3.1, engines node >=22.13, checked via `npm view` 2026-11-04)
- [sdk release notes] https://cursor.com/docs/release-notes/sdk
- [pricing] Cursor pricing: https://cursor.com/docs/account/pricing
- [cc-headless] Claude Code, Run programmatically: https://code.claude.com/docs/en/headless
- [cc-cli] Claude Code CLI reference: https://code.claude.com/docs/en/cli-reference
- [cc-sdk-ts] Claude Agent SDK TypeScript reference (SDKResultMessage): https://code.claude.com/docs/en/agent-sdk/typescript
