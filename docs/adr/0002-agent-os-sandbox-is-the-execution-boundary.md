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
