# Each Coding Agent's own OS sandbox is the execution boundary, and Cursor's escape gap is accepted

sous-chef runs Coding Agents on the developer's own machine, with no VM or firewall, and feeds them Work Item and PR text that may be hostile. To do real work, the agent has to run tests, builds and package scripts, and any of those can execute arbitrary code. So a list of allowed commands cannot be the boundary. What matters is what that code can *reach*. We confine every Run with the Coding Agent's own OS sandbox, which works the same way on any machine:

- **Writes:** only the worktree and the repo's `.git`.
- **Reads:** a fixed list of credential paths is never readable.
- **Network:** default deny, with an allowlist of package registries. GitHub and gist are explicitly denied.
- **Environment:** scrubbed, so no Tracker or GitHub tokens are in it.

Shell is then broadly allowed inside the sandbox. sous-chef keeps one sandbox policy, and each adapter translates it into its agent's own settings.

Claude Code's sandbox can be locked so the model cannot leave it. Cursor's cannot, as far as its docs say: the agent may re-run a blocked command outside the sandbox, gated only by an AI classifier that its docs admit can make mistakes. We accept that gap for v1. sous-chef prints a warning at the start of every Cursor Run and documents the gap, rather than drop Cursor or wrap it in a container.

## Considered Options

- **Command allowlist only.** Rejected. Package scripts and test runners make it trivially bypassable, and keeping the list right is a chore.
- **A container or VM per Run.** This is the strongest option, and the only real fix for Cursor's escape gap. Rejected for v1: it is heavy for a personal CLI, and macOS toolchains don't run in Linux containers. It can be added later behind the Coding Agent adapter seam if testing shows Cursor's sandbox can actually be escaped.
- **Claude-only unattended Runs.** Rejected, because v1 has to work with any Tracker and any Coding Agent.

## Consequences

- The agent never needs `gh` or git-over-SSH, because sous-chef owns pushes. That is why the sandbox's known macOS breakage of both doesn't matter.
- A scrubbed environment rules out the in-process `@cursor/sdk`, whose agents inherit sous-chef's own environment, tokens included. Cursor is therefore driven through the `agent` CLI as a separate process.
- The agent's built-in file and web tools run outside the sandbox, so they need separate permission rules:
  - file access stays inside the worktree;
  - WebSearch is off;
  - WebFetch only reaches domains the developer lists.

See [What guardrails bound a Run?](https://github.com/wkimdkim-nz/sous-chef/issues/10).

## Amendment: everything that runs repo code is sandboxed, including `checks`

The boundary above covers the agent, but sous-chef itself also runs repo code: the Recipe's `checks` commands, such as `npm test`, after each Pass. The agent may just have edited the scripts those commands run. If sous-chef ran them in its own process environment, which holds the Tracker and GitHub tokens, a hostile Work Item could make the agent rewrite a test script to steal a token, and sous-chef would then run it.

So **every command that runs repo code is sandboxed under the same sandbox policy and scrubbed environment as the agent**: the agent through its own sandbox, and `checks` through a sandbox that sous-chef's core owns, whichever Coding Agent the Run uses. Core uses Anthropic's standalone `@anthropic-ai/sandbox-runtime`, the library Claude Code's sandbox is built on. Core wraps each check command with it and spawns the result itself with the scrubbed environment, because the library doesn't scrub the environment.

### Considered Options

- **Run `checks` with only a scrubbed environment.** Rejected. The check could still read credential paths such as `~/.ssh` and reach github.com.
- **Run `checks` through the Coding Agent adapter**, as a `runCommand` using that agent's sandbox. Rejected. It puts something that isn't the agent's job behind the Coding Agent seam, and it would carry Cursor's best-effort gap over to `checks`.

### Consequences

- `@anthropic-ai/sandbox-runtime` is a 0.0.x beta, so its version is pinned. It needs Node ≥ 22.12, which becomes sous-chef's minimum Node version. On Linux it needs `bubblewrap` and `socat`, and on Ubuntu 24.04+ it needs an AppArmor setting changed. Adapter preflight checks for these.
- Tools that ignore proxy settings fail inside the sandbox even for allowed hosts: plain `ssh`, most database drivers, and `docker`. Jest's `watchman` integration also breaks, so a repo's `checks` may need `CI=true` or `--no-watchman`.
- Core now owns a sandbox runtime of its own. That makes it possible to wrap the Cursor CLI in it too, which might close the escape gap accepted above. [Can a headless Cursor Run escape its sandbox?](https://github.com/wkimdkim-nz/sous-chef/issues/14) weighs this.

See [What are the Tracker and Coding Agent adapter interfaces?](https://github.com/wkimdkim-nz/sous-chef/issues/12).

## Amendment: v1 drives Claude Code only, so Cursor's escape gap is deferred

v1 now supports one Coding Agent, Claude Code, and one Tracker, GitHub Issues. Cursor and Jira are deferred to a future enhancement behind the existing seams. That reverses the reason given above for rejecting **Claude-only unattended Runs**. In v1, every Run uses Claude Code's sandbox, which can be locked so the model can't leave it. v1 has no best-effort gap to accept and no per-Run Cursor warning.

The boundary itself is unchanged: one sandbox policy, translated by the Coding Agent adapter, plus the core-owned sandbox for `checks`. The Cursor parts of this ADR stand as groundwork for the enhancement:

- the accepted escape gap and its warning;
- driving Cursor through the `agent` CLI rather than `@cursor/sdk`;
- a container or VM, or wrapping the CLI in core's sandbox runtime, as the possible fix.

Whoever adds Cursor reopens this decision then. The probes in [Can a headless Cursor Run escape its sandbox?](https://github.com/wkimdkim-nz/sous-chef/issues/14), closed unresolved, are where that starts.

See the **Out of scope** section of [Wayfinder: sous-chef — tracker-driven AI development CLI](https://github.com/wkimdkim-nz/sous-chef/issues/1).

## Amendment: Workspace git never runs repo- or agent-controlled code

The boundary lets the agent write the repo's `.git`, because committing needs objects, refs and the index. But `.git` also holds code that git runs: `hooks/*`, plus whatever `.git/config` or a worktree's `config.worktree` names in `core.hooksPath`, `core.fsmonitor` or a filter driver. sous-chef's Workspace runs `git` (fetch, merge, push) **outside** the sandbox, with the push credential in reach. So without a guard, an agent steered by a hostile Work Item could plant a hook that sous-chef itself would run, unsandboxed, on its next git command.

So **Workspace git never runs code the repo or the agent controls**, and sous-chef works in **a clone it owns**, never in the developer's own clone:

- Every Workspace git command runs with `-c core.hooksPath=/dev/null -c core.fsmonitor=false -c protocol.ext.allow=never`, with `GIT_CONFIG_NOSYSTEM=1` and `GIT_CONFIG_GLOBAL=/dev/null`, so only sous-chef's own settings apply.
- The sandbox denies writes to `.git/hooks/**`, `.git/config` and `.git/worktrees/*/config.worktree`.
- Worktrees come from a partial clone (`--filter=blob:none`) under `$XDG_DATA_HOME/sous-chef/<owner>/<repo>/`, created by `init` or on first start.
- The GitHub token reaches git only through an inline credential helper, set per command after clearing every configured helper, and only in that one git child's env. It's never written to disk or git config.

### Considered Options

- **Use the developer's own clone.** Rejected. The guard above protects only sous-chef's git calls. A hook or config the agent planted in that `.git` would still run the next time the developer runs `git commit`, unsandboxed and with all their credentials. The agent could also move the developer's local refs. The developer's own git setup (husky, sparse checkout, a dirty tree) would leak into Runs as well. Using its own clone costs disk space and the wait for a first clone, which the partial clone keeps small.
- **Guard only with flags, or only with sandbox write denies.** Rejected. Each is one mistake away from failing: a git command missing a flag, or a config path the deny list didn't name. Together, both would have to fail.

### Consequences

- Git LFS and submodules don't work in v1, because global config (where LFS installs itself) is ignored and the agent can't reach github.com. Preflight warns when the repo uses either. If a Run's diff adds a file matching an LFS pattern, the PR isn't marked ready, and the Run goes to Needs Human.
- To try a Run's branch locally, the developer fetches it into their own clone. It's always pushed.
- The sandbox probe suite gains a probe that plants a hook and checks it never fires.

See [What CLI commands and config file does sous-chef have?](https://github.com/wkimdkim-nz/sous-chef/issues/15).

## Amendment: each Run gets its own clone, and unsandboxed git only touches the trusted clone

The second amendment guarded a gitdir that the agent shares with sous-chef, using three `-c` flags and three write denies. A probe on git 2.52 got past both. The agent rewrote `worktrees/<key>/commondir` (or its worktree's `.git` file) to point at a `config` of its own. sous-chef's unsandboxed `git merge` then ran a merge driver from that config, and `git remote get-url --push origin` returned an attacker's URL, which the inline helper would have handed the token to. The shared gitdir also let one Run's agent move another Run's refs. A deny list cannot win this: git reads executable config from too many keys (`filter.*`, `diff.*.textconv`, `core.sshCommand`, `gpg.program`, `core.editor`, merge drivers), and too many paths decide where that config lives.

So **git that runs outside the sandbox only ever touches sous-chef's own repository and config**:

- **Trusted clone.** One full clone per repo under `$XDG_DATA_HOME/sous-chef/<owner>/<repo>/`. It is the only repository with network or credentials, and no agent or check can write to it. It fetches and pushes by the literal URL `https://github.com/<owner>/<repo>.git`, never by remote name. The inline credential helper is bound to `credential.https://github.com.helper`. `transfer.fsckObjects=true` is set, `gc.auto=0` is set, maintenance is off, and it never prunes.
- **Run clone.** Each Run gets an ordinary clone in its own Run directory, made with `git clone --shared` from the trusted clone, so it reads trusted objects through alternates and writes only its own. The sandbox lets a Run write only its Run directory and temp. The trusted clone and other Runs' directories are readable but not writable. The trusted clone holds no secrets.
- **Every git command against a Run clone runs in the core sandbox**, under the same policy as `checks`: checkout, the inbound fetch, the base merge, status and unsaved-work checks, and bundling. Anything the agent planted in that clone's config can only run there.
- **Inbound**, a sandboxed `git fetch <trusted-clone-path> <ref>` runs inside the Run clone. `upload-pack` runs against the trusted repository, so it reads only trusted config.
- **Outbound**, a sandboxed `git bundle create` writes `<last-imported>..refs/heads/<branch>` to a file in the Run directory. sous-chef then runs `git fetch <bundle> refs/heads/<branch>:refs/sous-chef/incoming/<key>` in the trusted clone. Fetching a bundle reads one data file, so no foreign config is involved and nothing is executed.
- **Before anything is pushed**, the import is checked in the trusted clone. Only the one named ref is taken, its objects pass fsck, the new tip descends from the last pushed tip (or from the base on the first push), and the protected-path and LFS-pattern checks run on `base..tip` with `--no-ext-diff --no-textconv`. Any failure means nothing is pushed, and the Run goes to Needs Human with the reason. There is no size cap, because the agent can already fill the disk from its own worktree.
- **The base merge** happens in the Run clone, sandboxed, after an inbound fetch of the base. A conflict becomes a Feedback Round, as already decided.
- The `-c` flags and env isolation from the second amendment stay on every git command sous-chef runs, sandboxed or not. The three `.git` write denies are dropped, since no unsandboxed git ever reads a Run clone.

### Considered Options

- **Keep the shared gitdir and pin it**, with explicit `GIT_DIR`/`GIT_COMMON_DIR`, a literal push URL and a host-bound helper. Rejected. That fixes the two probed paths, but every unsandboxed git command would still read config the agent can write, which is the "a config path the deny list didn't name" failure again.
- **Widen the deny list.** Rejected for the same reason, and `commondir` redirection steps around it anyway.
- **No git for the agent**, with sous-chef committing snapshots of the worktree. Rejected. It's the safest, but it throws away the agent's commits and messages, and leaves the agent no `diff`, `log` or `stash` against a base.
- **An unsandboxed `git fetch <run-clone-path>`** to import. Rejected. Its `upload-pack` reads the Run clone's config.
- **Hardlinked Run clones** (`git clone --local`). Rejected. The agent owns the files and could rewrite a shared object in place, corrupting the trusted clone.
- **A partial (`blob:none`) trusted clone**, as the second amendment chose. Rejected. Run clones have no network, so they couldn't fetch history blobs lazily, and `git log -p`, `blame` and old diffs would fail for the agent. A full clone is paid once per repo, and alternates keep each Run clone small.

### Consequences

- The first clone of a repo downloads its full history.
- The trusted clone only grows, because it never prunes while Run clones depend on its objects through alternates. Repacking without pruning is safe.
- Unsaved work in a Run clone is a dirty tree or commits not yet imported. It's found by sandboxed `status`/`rev-list`. The agent can lie in that output, but then the only thing it can lose is its own work.
- The last-imported tip is the remote branch tip, which is already the record (ADR 0003), so the local record gains nothing. A crash between import and push redoes the push, which the incoming ref in the trusted clone shows. A lost Run clone is recreated with `clone --shared` and a checkout of the branch.
- The sandbox probe suite's single hook probe is replaced by five probes. Each one has a planted payload try to write a marker outside the Run directory, and fails if the marker appears:
  1. Planted config never runs outside the sandbox: hooks, `core.hooksPath`, `core.fsmonitor`, `core.sshCommand` with `url.*.insteadOf`, `gpg.program`, `core.editor`, `filter.*`, `diff.*.textconv` and a merge driver set up through `.gitattributes`. The probe drives the whole Workspace sequence: inbound fetch, base merge, bundle, import, push.
  2. Gitdir redirection: the agent points its `.git`, `commondir` and `objects/info/alternates` at paths it controls. Writes stay in the Run directory, and the import takes only the named ref.
  3. Cross-Run writes: one Run's agent can't write another Run's directory or the trusted clone's refs, objects or config.
  4. Import rejection: a bundle with an extra ref, a tip that doesn't descend from the last pushed tip, or a corrupt object goes to Needs Human with nothing pushed.
  5. Helper scoping: `git credential fill` for any host but github.com returns no token.

See [How does Workspace git stay safe when the Coding Agent can write the gitdir?](https://github.com/wkimdkim-nz/sous-chef/issues/28).

## Amendment: a protected-path or LFS-pattern change is held until a human releases it

The third amendment's import check stops a protected-path or LFS-pattern change from being pushed. But the earlier decisions still pushed such commits at several points: the draft PR, a Feedback Round ("push, then convert to draft") and a limit hit ("push the commits that exist"). The Run branch lives in the same repository, so GitHub Actions runs a pushed `.github/**` change with the repo's secrets and a write-capable `GITHUB_TOKEN`, before the developer has seen it. Only a Withdraw held such commits back. A Coding Agent steered by hostile text could use any of the other push points to leak CI secrets.

So **no push ever carries an unreviewed protected-path or LFS-pattern change**:

- **One gate, every push point.** The draft PR, Self-Review fixes, a Feedback Round, a limit hit, a Withdraw and a base merge all go through the import check. A path is flagged if it matches a protected path or is a newly added file matching an LFS pattern, **and** its content at the new tip differs from both the last pushed tip (on a first push, the base) and the current base. This replaces the third amendment's `base..tip`: changes the developer has already pushed pass, files a base merge brought in unchanged pass, and an edit the agent has backed out passes. A protected file resolved differently from base in a merge is flagged.
- **A tripped gate makes a Hold.** Nothing is pushed. The commits stay in the Run clone and at `refs/sous-chef/incoming/<key>` in the trusted clone, which is never pruned while held. The Run goes to Needs Human. On a first push, no branch or PR is created. In a Feedback Round, the PR is converted to draft and no thread replies are posted. A limit hit no longer pushes held commits. On a Withdraw, nothing is written to the Tracker beyond the Status Comment.
- **The default protected paths widen from `.github/workflows/**` to `.github/**`**, because workflows run local composite actions from `.github/actions/**` with the same secrets.
- **Only a human at a terminal releases a Hold.** `sous-chef review <key>` requires a TTY and has no `--yes`. It shows the full held diff in a pager and offers Accept, Reject, Discard or Quit. It never checks out the held tip, so agent code is never one keystroke from running unsandboxed. `review` writes nothing to git or the Tracker. It writes a **decision file**, outside the agent's reach, naming the held tip and the last pushed tip. `start` applies it on its next tick only if `refs/sous-chef/incoming/<key>` still equals the held tip, the remote branch still equals the last pushed tip, and no Pass is running. Otherwise the decision is stale: it's discarded, and the developer is notified. Accept is the gate's one exception, and it pushes exactly that SHA.
  - After a Needs Human Hold, `start` moves the Work Item straight back to In Dev or In Review. **Accept** pushes and continues. **Reject** posts the developer's text as a comment and continues from the held commits. **Discard** resets to the last pushed tip (or the base) and re-runs the held step, with limits still counting.
  - After a Withdraw Hold, only Accept (push only) and Discard are offered, and the Run is never restarted.
- **The developer can still act outside `review`.** A Trusted User comment plus Ready is a Reject. Pushing to the Run branch themselves follows the re-Ready rules: a remote at or descending from the held tip continues from it, an unchanged remote continues from the Hold, and a diverged remote wins and discards the Hold.
- **The Status Comment says what to do, never how.** It shows the reason, the push point, the held tip's short SHA, and file paths with +/- counts, and points to `review`. It shows no diff, because an agent could have written something it read into a held file, and no local paths. Every Hold and every stale decision sends a notification, including after a Withdraw.

### Considered Options

- **Push, then keep the PR a draft** (the earlier rule). Rejected. A draft PR doesn't stop Actions: CI runs on the pushed branch before anyone has looked at it.
- **Check `base..tip`**, as the third amendment wrote. Rejected. Once a protected path was touched, every later push would be held, even after the developer accepted the change, and a base merge would flag base's own workflow edits.
- **The developer pushes the held commits by hand**, with copy-pasted git commands. It needs no gate exception, but it means five or more error-prone steps and a local clone, and the review step is easy to skip. It stays possible as the advanced path.
- **A release label** (`agent:allow-protected`) on the Work Item. Rejected. It approves a change the developer can only see in a public comment, and that comment can't safely include the diff, so it invites rubber-stamping the riskiest push.
- **Require a token without the `workflow` scope**, so GitHub itself rejects workflow pushes. Not required: it covers only `.github/workflows/**` and breaks the zero-setup `gh auth token` path. `doctor` and the banner warn when the token has it, as defense in depth.

### Consequences

- `start` stays the only writer. "No command acts on a Run" narrows to "only `start` writes, and `review` only queues a decision from a human at a terminal". `status` takes an optional `<key>` and shows Holds, decisions, and the fetch command for the advanced path.
- Accept is new code on the riskiest path. Binding it to the exact held tip and last pushed tip is what keeps it narrow. The crash matrix covers applying a decision, and the import-rejection probe gains a case: a decision whose held tip has moved pushes nothing.
- Other code that CI runs (test scripts, `package.json` scripts, Makefiles) is still pushed before review. That's an accepted, documented risk: keep secrets out of jobs that branch pushes and PRs trigger, and gate secret-using jobs behind an Environment with required reviewers.

See [What gates a push, as opposed to marking the PR ready?](https://github.com/wkimdkim-nz/sous-chef/issues/20).

## Amendment: reads are an allowlist, and outgoing content is scanned as a backstop

The read policy denied a fixed list of credential paths and left the rest of `$HOME` readable: `~/.git-credentials`, `~/.claude`, shell history, cloud CLI config, `.env` files in other projects. The network policy keeps the agent off GitHub, but sous-chef then carries the agent's commits and text there itself, as the developer. Claude Code's own Read, Grep and Glob tools don't obey the sandbox's `denyRead` at all. A Coding Agent steered by hostile text could read a secret and leave it in a commit or a PR body.

No outgoing scan can stop an agent that encodes what it read (base64, split across commits, paraphrased). So **what the agent can't read can't leave is the guarantee, and the scan is only a backstop for accidents**:

- **Reads are an allowlist, enforced in three places.** The agent's shell runs under Claude Code's sandbox with `permissions.blockReadsOutsideWorkingDirectories` plus `sandbox.filesystem.allowRead` for toolchain directories. The same switch hard-refuses Read, Grep, Glob and LSP outside the Run clone in every permission mode. `checks` and sandboxed git run under the core srt sandbox with `denyRead: ["~/"]` and the same `allowRead`. The old credential list stays as narrower denies, which beat a broader allow, so `~/.cargo/credentials.toml` stays unreadable inside an allowed `~/.cargo`.
- **The allowlist.** Built-in toolchain defaults (nvm, fnm, Volta, pnpm, `.rustup`, `.cargo`, `.pyenv`, `.m2`, `.gradle`), the Run's own directory and the trusted clone. Other Runs' directories are no longer readable. `~/.npmrc`, `~/.pypirc`, `~/.gitconfig`, `~/.git-credentials` and `~/.docker/config.json` are always denied, because they often hold tokens. The developer extends the list only with `sandbox.allowRead` in the per-repo user config, never from the repo or the Recipe, and an entry covering an always-denied path is fatal at startup.
- **Versions.** sous-chef requires Claude Code ≥ 2.1.257 and `@anthropic-ai/sandbox-runtime` ≥ 0.0.83, and `doctor` checks both.
- **Everything that leaves is scanned.** Before a push, the content the import gate already flags as new (differing from both the last pushed tip and the base) is scanned. Before a post, so are the PR title and body, thread replies, round summaries, needs-info questions and any agent text quoted in the Status Comment. The rules are a bundled, offline, gitleaks-style set, plus an exact match on every credential sous-chef holds (the GitHub token and the Anthropic key, raw, base64 and URL-encoded). The scan is always on. False positives are allowed by rule ID or path glob in `secretScan.allow`, in the user config only. Public and private repos get the same policy. On a public repo, `doctor`, the banner and Hold notifications say that pushes are world-readable at once.
- **A hit is a Hold.** Commits are held exactly as a protected-path change is held. Text is withheld, not posted. The Status Comment shows the reason, the rule ID and the paths, never the matched value. `sous-chef review <key>` shows the held diff or text locally and offers Accept, Reject or Discard. An exact match on a credential sous-chef holds offers only Reject or Discard, and the notification says to rotate that credential.

### Considered Options

- **Keep the denylist and lengthen it.** Rejected. Every developer's `$HOME` holds secrets the list doesn't name, and the list would never be finished.
- **Make the scan the guarantee** and leave reads broad. Rejected. An agent that encodes what it read gets past any scanner.
- **No scan at all.** Rejected. Real repos sometimes carry a real secret in a fixture or `.env.example`, and catching that is cheap.
- **Block registry uploads** with a TLS-intercepting srt proxy that allows only GET and HEAD, with Claude Code pointed at it via `httpProxyPort`. Deferred. srt ≥ 0.0.51 can do it (`tlsTerminate` plus `filterRequest`, with `refuseOpaqueTunnels`), but every toolchain in a Run would have to trust sous-chef's CA, each configured differently, and that is the likeliest thing to break on a real repo.

### Consequences

- Toolchains that read settings from an always-denied file, or write caches under an allowed directory (on Linux, an `allowWrite` nested inside an `allowRead` carve-out is silently read-only, anthropics/sandbox-runtime#446), need configuration sous-chef writes into the Run directory. That is decided with what a real repository needs writable.
- The probe suite gains cases: Bash, the Read tool and a `checks` command each fail to read a planted `~/.aws/credentials`; one Run can't read another Run's directory; a planted secret in a commit and in a PR body each make a Hold; the GitHub token in base64 matches exactly.
- **Accepted, documented risk:** registries accept uploads, so an agent steered by hostile text could publish a private repo's source with an attacker's own token. The developer's own publish tokens are unreadable, and only Trusted Users start Runs.

See [What can a Run read, and what may leave through the commits and text sous-chef publishes?](https://github.com/wkimdkim-nz/sous-chef/issues/21).

## Amendment: a Run loads no Claude Code config from the host or the repo, only pinned instructions and skills

The sandbox wraps only shell commands. Claude Code's command hooks, stdio MCP servers, plugin monitors, LSP servers and WebFetch all run outside it, with the developer's full access. Leaving `settingSources` out loads the developer's `~/.claude` (hooks, MCP servers signed in to other services, plugins, memory) and the repo's `.claude/settings.json` hooks and `.mcp.json`. That is code running unsandboxed while the agent reads text that may be hostile. The Run's own commits could also change the repo's agent config before a later Pass loads it. And the test suites ran with `settingSources: []`, so they never saw what production loaded.

So **a Run is built from sous-chef config plus the base branch, never from what happens to be on the machine or the Run's branch**:

- **Isolation.** Every Pass runs with a fresh `CLAUDE_CONFIG_DIR` inside the Run directory (holding only the API key), `settingSources: []`, `settings: { disableAllHooks, disableClaudeAiConnectors, permissions.blockReadsOutsideWorkingDirectories, permissions.blockWritesOutsideWorkingDirectories }`, `strictMcpConfig: true` with an empty `mcpServers`, and `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`.
- **Instructions.** sous-chef reads `CLAUDE.md`, `AGENTS.md` and `.claude/rules` from the base-branch tip in the trusted clone at Run start and appends them through `systemPrompt: {type: 'preset', append}`. The repo's settings, hooks and `.mcp.json` never load.
- **Skills.** Only the skills the Recipe names with `/name` load. They are copied at Run start from `~/.claude/skills` or the repo's `.claude/skills` at base into a skills-only local plugin in the Run directory, where their scripts can run in the sandbox. A Recipe skill that is missing, exists in both places, or carries frontmatter `hooks` or a `.claude-plugin/` folder is a preflight error. `!` lines stay, because they run through permission-checked, sandboxed Bash.
- **Pinning.** Instructions and skills are pinned for the whole Run, as the Recipe is, so the Run's own commits never change what its later Passes load. No pre-Pass check of agent-config paths is needed.
- **Tools.** `dontAsk` permission mode, the explicit tool allowlist, `disallowedTools: ["mcp__*"]`, and no `canUseTool` that approves. sous-chef's own in-process `PreToolUse` hook refuses WebFetch outside `docsDomains`, because Claude Code pre-approves some documentation domains on its own.
- **No MCP servers in a Run in v1.**
- **Managed policy.** `start` refuses if endpoint managed settings or `managed-mcp.json` force hooks or MCP servers, because the Run couldn't be confined. Other managed policy is a banner warning. Server-managed settings don't reach a Console API key, which is documented.
- **One options builder.** Production, the contract suites and the probes build `query()` options through the same function.
- **Versions.** Claude Code ≥ 2.1.314, for `blockWritesOutsideWorkingDirectories` and the symlink fixes before it.

### Considered Options

- **Load everything, as an interactive session would.** Rejected. Hooks and MCP servers would run unsandboxed, and a Run would depend on whatever is in the developer's home directory that day.
- **Load the `project` source** to get the repo's `CLAUDE.md` and skills. Rejected. The same switch loads `.claude/settings.json` hooks, from the Run's own working tree.
- **Point a local plugin at the repo's `.claude/`.** Rejected. A plugin root also auto-loads `hooks/hooks.json` and `.mcp.json`.
- **Check before every Pass that agent-config paths match base.** Not needed once nothing is loaded from the Run clone.
- **Make `.claude/**` and `.mcp.json` protected paths.** Rejected as a default. A protected path is one that runs before human review, as CI does. These run only when someone opens Claude Code on the branch, like `package.json` scripts. They are documented as sensible extra `protectedPaths` entries.

### Consequences

- Repos that rely on project hooks or MCP servers lose them in Runs. A user skill that reaches back into `~/.claude/skills` by absolute path breaks, which is documented.
- The probe suite gains cases: Read and Grep through a symlink to a planted `~/.aws/credentials` are refused; Edit and Write outside the Run clone and through a symlink are refused; WebFetch to a domain outside `docsDomains`, including a pre-approved docs domain, is refused; a planted user hook, project hook, user MCP server and `.mcp.json` server never fire; a skill's `!` line can't reach the network; a skill with frontmatter `hooks` fails preflight; the session's reported tools, MCP servers and skills match the expected set exactly, with nothing from the developer's real `~/.claude`.

See [Which Claude Code settings, hooks and MCP servers load in a Run?](https://github.com/wkimdkim-nz/sous-chef/issues/22).
