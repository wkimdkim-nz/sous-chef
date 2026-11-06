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
