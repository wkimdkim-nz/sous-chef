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
