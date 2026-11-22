# Sandbox read policy: allowlisting reads, toolchains, network filtering, built-in tools

Research for [ADR 0002](../adr/0002-agent-os-sandbox-is-the-execution-boundary.md), whose policy today says "Reads: a fixed list of credential paths is never readable" (a denylist).

**Question:** Can sous-chef make filesystem reads an allowlist instead, what do toolchains need under `$HOME` if so, can the network proxy filter by HTTP method or path, and does the read policy cover Claude Code's own file tools?

**As of:** 2026-11-22. Versions checked: Claude Code CHANGELOG up to 2.1.330, npm `@anthropic-ai/claude-agent-sdk` 0.3.330 (`sdk.d.ts`), `@anthropic-ai/sandbox-runtime` 0.0.83 (source at `anthropics/sandbox-runtime` commit `0bbebed`, 2026-11-12). Both packages ship often, so pin them and re-check.

## TL;DR

1. **Read allowlisting works, as deny-then-re-allow.** Both Claude Code and sandbox-runtime default to read-everything, but `denyRead: ["~/"]` (or `/Users`, `/home`) plus `allowRead: [worktree, …]` gives workspace-only reads. An `allowRead` beats a broader `denyRead`, and a narrower `denyRead` still beats a broader `allowRead`. Writes work the other way round: `denyWrite` always wins. [S1][S2][S3]
2. **Toolchains under `$HOME` break unless they are re-allowed one by one.** On macOS and Linux nothing is documented for this beyond git config. The only list Anthropic publishes is sandbox-runtime's Windows profile grant, which is a usable model. [S4][S5]
3. **The network is filtered by host name only, unless you use the library.** Claude Code's settings can't filter by method or path. sandbox-runtime ≥ 0.0.51, used as a library, can do it with `tlsTerminate` plus a `filterRequest` callback that sees the whole decrypted request. [S1][S6]
4. **Claude Code's file tools don't honour `sandbox.filesystem.denyRead`.** They follow `Read(...)` permission rules, which are best-effort for Grep and Glob, or `permissions.blockReadsOutsideWorkingDirectories`, which is a hard refusal. [S1][S7][S3]

## 1. Allowlisting reads

**Claude Code** (`sandbox.filesystem.*`): `allowRead` was added in v2.1.77 to "re-allow read access within `denyRead` regions". [S8] The docs give this precedence:

- `denyRead ["~/"]` with `allowRead ["~/projects"]`: only `~/projects` is readable.
- `allowRead ["~/"]` with `denyRead ["~/.env"]` or `["~/**/.env"]`: the deny holds. [S1]

The SDK's `Options.sandbox.filesystem` accepts `allowRead` and the managed-only `allowManagedReadPathsOnly` in the 0.3.330 types, although the SDK reference page still lists only `allowWrite`, `denyWrite` and `denyRead`. [S9][S10] Some gotchas:

- Paths use `~/` and absolute paths, unlike the `//` prefix of permission rules.
- On Linux and WSL2, an entry for a path that doesn't exist yet has no effect.
- Arrays merge across settings scopes. [S1]

An easier route is `permissions.blockReadsOutsideWorkingDirectories: true` (v2.1.257+). With sandboxing on, it denies sandboxed commands reads of `$HOME`, `/Users`, `/home`, `/root`, `/Volumes`, `/mnt`, `/media`, `/run/media` and `/srv`. It then re-opens:

- the working directories and session worktrees;
- the session temp directory;
- the parts of `~/.claude` that commands need;
- `~/.gitconfig` and `~/.config/git/{config,ignore,attributes}`.

`~/.git-credentials` stays blocked. [S3]

**sandbox-runtime** (`filesystem.denyRead` / `allowRead`): `allowRead` was added in commit `75944d8` on 2026-03-08 and first shipped in npm 0.0.42 (0.0.40 and 0.0.41 lack it). The precedence is the same as Claude Code's. [S2] How each platform enforces it:

- **macOS (Seatbelt):** an ordered profile. First `(allow file-read*)`, then `(deny …)` for the denyRead entries, then `(allow …)` for the allowRead ones, then the narrower denies emitted again so the allows can't clobber them. The last matching rule wins. [S11]
- **Linux (bubblewrap):** a denied directory becomes an **empty writable tmpfs**, so writes into it vanish when the command exits. `allowRead` paths are bind-mounted back over it. Globs are expanded once, when the command is wrapped, so files created later aren't covered. A path that doesn't exist at wrap time is not denied, and a pattern whose first component is a wildcard is skipped. [S2] Open bug [#446](https://github.com/anthropics/sandbox-runtime/issues/446): an `allowWrite` nested inside an `allowRead` carve-out under a `denyRead` is silently read-only. That matters for cache writes such as `~/.npm`.
- `~/.npm/_logs` is writable by default, but stops being writable under a home read-deny unless you re-allow it. [S2]

## 2. What toolchains need under `$HOME`

Anthropic documents no macOS or Linux list. sandbox-runtime's Windows read grant is the closest guidance. [S4] It re-opens these, read-only:

- npm global installs, nvm, fnm, Volta and pnpm;
- `.rustup` and `.cargo`, with `.cargo/credentials` and `credentials.toml` kept denied;
- `.pyenv`, pip config and Python user site-packages;
- `.m2`, `.gradle`, `go`, `.bun`, `.deno`, `.dotnet`, `.nuget`, `pipx`, `miniconda3` and `anaconda3`;
- `.npmrc` and `.gitconfig`.

It also keeps `.netrc`, `.pypirc` and similar files denied.

For macOS and Linux, the equivalent default locations to re-allow are the following. These are each tool's own defaults, not Anthropic guidance.

- **Read:** `~/.nvm`, `~/.volta`, fnm's directory (`~/.local/share/fnm`, or `~/Library/Application Support/fnm` on macOS), `~/.rustup`, `~/.cargo/{bin,registry,git}`, `~/.pyenv`.
- **Read and write:** the caches, `~/.npm`, `~/.cache` (pip, uv and others; `~/Library/Caches` on macOS) and `~/.cargo/registry`.

Homebrew (`/opt/homebrew`, `/usr/local`) and system paths sit outside `$HOME`, so they're unaffected. Claude Code's troubleshooting section lists other breakage: watchman, docker, Go TLS on macOS, ssh git. Nothing there is specific to a home read-deny. [S1]

## 3. Network filtering

- **Claude Code:** "The built-in proxy enforces the allowlist based on the requested hostname and, by default, does not terminate or inspect TLS traffic." [S1]
  - `sandbox.network.tlsTerminate` (user or managed scope, experimental) terminates TLS for credential masking. No setting filters by method or path. [S3][S10]
  - `httpProxyPort` and `socksProxyPort` hand all filtering to your own proxy, and Claude Code's domain lists then stop applying. [S1]
- **sandbox-runtime** (library only):
  - `network.tlsTerminate` (≥ 0.0.51) is a MITM with an ephemeral or supplied CA.
  - `network.filterRequest(request, info)` gets a web `Request` (method, URL, headers, body stream) and returns allow or deny. A throw fails closed. [S6][S2] So "GET to registry.npmjs.org, but not PUT" is expressible.
  - Pair it with `network.refuseOpaqueTunnels`, so that SOCKS connections and tunnels that aren't terminated can't bypass the filter.
  - Certificate-pinning and mTLS clients need `excludeDomains`, and those domains are then filtered by host only.
  - `network.mitmProxy` (≥ 0.0.38) routes chosen domains to an upstream proxy over a Unix socket. [S2]
  - The README warns about domain fronting. [S2]

## 4. Claude Code's own file tools

- "A `denyRead` entry doesn't stop the Read tool, and `allowedDomains` doesn't limit WebFetch." [S1]
- `Read(path)` deny rules cover Read. Claude makes a "best-effort attempt" to apply them to Grep, Glob, @-mentions and IDE context.
- `Grep(...)` and `Glob(...)` path rules are accepted but never match, so write `Read(...)` rules instead.
- `Read` and `Edit` rules are merged into the sandbox config, so a `Read(~/.aws/**)` deny also covers Bash. [S7]
- `blockReadsOutsideWorkingDirectories` is a hard refusal of Read, Grep, Glob and LSP outside the working directories in every mode, including `bypassPermissions`. [S3]

For sous-chef, the strongest combination for the agent is `blockReadsOutsideWorkingDirectories` plus explicit `Read` denies for credential paths plus `allowRead` entries for toolchains.

## Sources

- [S1] Claude Code, Sandboxing: https://code.claude.com/docs/en/sandboxing
- [S2] sandbox-runtime README (0.0.83): https://github.com/anthropics/sandbox-runtime/blob/main/README.md
- [S3] Claude Code settings reference (`permissions.blockReadsOutsideWorkingDirectories`, `sandbox.filesystem.allowRead`, `sandbox.network.tlsTerminate`): https://code.claude.com/docs/en/settings-reference
- [S4] sandbox-runtime README, "Reading your user profile" / "What is readable" (Windows)
- [S5] Claude Code settings reference, "Sandboxed commands under the block"
- [S6] `src/sandbox/request-filter.ts` (`FilterRequestCallback`), commit `84e22b5`, first tag v0.0.51: https://github.com/anthropics/sandbox-runtime/blob/main/src/sandbox/request-filter.ts
- [S7] Claude Code, Permissions (Read/Edit rules; "How permissions interact with sandboxing"): https://code.claude.com/docs/en/permissions
- [S8] Claude Code CHANGELOG (2.1.77 `allowRead`; 2.1.224 trailing-slash deny fix; 2.1.257 read block): https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
- [S9] Agent SDK TypeScript reference, `SandboxFilesystemConfig`: https://code.claude.com/docs/en/agent-sdk/typescript
- [S10] `@anthropic-ai/claude-agent-sdk` 0.3.330 `sdk.d.ts`, sandbox settings schema
- [S11] `src/sandbox/macos-sandbox-utils.ts`, read-rule ordering comment: https://github.com/anthropics/sandbox-runtime/blob/main/src/sandbox/macos-sandbox-utils.ts
