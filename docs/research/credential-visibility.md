# Credential visibility: which processes can see the Anthropic key and the push token

Research for [Where can each credential be seen, and by what?](https://github.com/wkimdkim-nz/sous-chef/issues/24) and [ADR 0002](../adr/0002-agent-os-sandbox-is-the-execution-boundary.md).

**Question:** Can a sandboxed command (the agent's Bash, a `checks` command, sandboxed git) read a credential that sits in another process's environment or memory: sous-chef's own Node process, the Claude Code CLI, or the unsandboxed git child that pushes? Which ways of handing over a credential keep it out of every environment?

**As of:** 2026-11-22. Versions checked: macOS with SIP on, XNU `main` source; `@anthropic-ai/sandbox-runtime` 0.0.83 (source `anthropics/sandbox-runtime` commit `0bbebed`, and the npm build); Claude Code 2.1.274 (the installed native binary, for strings and behaviour) plus the docs at code.claude.com; `@anthropic-ai/claude-agent-sdk` 0.3.330; git 2.52.0. Experiments used only fake values (`FAKE_SECRET=…`, `sk-ant-api03-FAKE…`) in processes started with `env -i`, and local HTTP servers on 127.0.0.1. No real credential was read.

## TL;DR

1. **macOS: the environment a process started with is readable by every same-uid process, including from inside the sandbox.** `ps eww` and `sysctl(KERN_PROCARGS2)` read it from the target's stack. The only kernel check is the same uid. Env vars are left out only when the target is `CS_RESTRICT`: Apple platform binaries, or a process that set the flag on itself. Deleting a var from Node's `process.env` changes nothing, because the original block stays in memory. [S1][S2][S3] Probed: **inside srt's Seatbelt sandbox, `KERN_PROCARGS2` on an unsandboxed Node process and on an unsandboxed Claude Code 2.1.274 process returned their full environments, including a var Node had deleted.** srt's profile doesn't list `kern.procargs2` in its `sysctl-read` allowlist, yet the call succeeded. The same profile does block an unlisted sysctl (`kern.boottime`), so Seatbelt apparently doesn't gate procargs through `sysctl-read`. The `process-info* (target same-sandbox)` rule didn't stop `proc_pidpath` or `proc_listpids` either. [S4]
2. **So on macOS, ADR 0002's "only in that one git child's env" and "Anthropic key in Claude Code's env" are readable by any agent command or check of any Run**, for as long as the process lives. Claude Code's own Bash sandbox ships the same profile text (strings in the 2.1.274 binary match), so the agent's shell almost certainly sees the same. That wasn't probed end to end, because it needs a model turn. `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` and `sandbox.credentials` only change what *children* inherit. Neither hides the parent's initial environment. [S5][S6]
3. **Linux: srt always runs bwrap with `--unshare-pid --unshare-user --proc /proc`**, unless `enableWeakerNestedSandbox` is set. A sandboxed command sees only its own PID namespace, so other processes' `/proc/<pid>/environ`, `mem` and `fd` don't exist for it. `ptrace` doesn't matter from inside. It does matter for unsandboxed same-uid code: `environ` and `fd` need only `PTRACE_MODE_READ` (same uid and dumpable), and Yama blocks only `ATTACH`-mode access such as `mem`. [S7][S8][S9][S10]
4. **Claude Code can take the API key without any environment variable.** `apiKeyHelper` (documented) runs outside the sandbox through `/bin/sh` and is cached in memory for 5 minutes, and its output wasn't in hook subprocess environments. `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR=<n>` is undocumented but present in 2.1.274. Probed: the CLI read the key from fd 3, sent it as `x-api-key`, deleted the variable, and didn't pass the fd or the key to hooks. With an env key and a fresh `CLAUDE_CONFIG_DIR`, `-p` wrote nothing key-related to the config dir: no `customApiKeyResponses` and no `primaryApiKey`. [S11][S12][S13]
5. **git: an inline helper that reads an inherited fd works, and the token never enters argv or env.** Probed: fd 3 survived through `git` → `git-remote-http` → `sh` → helper, and the server received the token. git, its re-exec and `git-remote-http` had no token in their argv or environment. git doesn't close inherited fds. It only marks its own fds close-on-exec. Credentials travel to and from helpers over stdin and stdout. `-c` values are re-exported to children in `GIT_CONFIG_PARAMETERS`, so the helper string must not contain the token. [S14][S15][S16]
6. **The SDK passes `options.env` as the CLI's process environment** (`child_process.spawn(..., { env })`, defaulting to `{...process.env}`). On macOS, whatever goes there is readable through `KERN_PROCARGS2`. `settings` goes to `--settings`, which puts it in argv, which is always readable. `spawnClaudeCodeProcess` lets the caller add an extra stdio fd. [S17]

## 1. macOS: reading another process's initial environment

**Mechanism.** `ps -E/-e` "Display the environment as well. This does not reflect changes in the environment after process launch." [S1] `ps` gets it from `sysctl({CTL_KERN, KERN_PROCARGS2, pid})`. In XNU, `sysctl_procargsx()`:

- copies the top of the target's user stack, where `execve` placed argv and envp, through `vm_map_copyin` on the target's map. So it reads **live memory**, not a saved copy. [S2]
- refuses only when the caller's uid ≠ target's uid and the caller isn't root. [S2]
- strips env vars (`omit_env_vars`) unless any of these holds: the target is the caller itself, the target is **not** `cs_restricted`, SIP allows unrestricted dtrace (SIP off), or the caller has an entitlement. A DEVELOPMENT/DEBUG kernel never strips them. [S2]

`cs_restricted(p)` is `csflags & CS_RESTRICT`. [S3] The hardened runtime (`CS_RUNTIME`) is a different flag. The Claude Code binary is signed `flags=0x10000(runtime)` and still exposed its environment in the probe. A process can set `CS_RESTRICT` on itself with `csops(getpid(), CS_OPS_SET_STATUS, &flags)`, since `CS_RESTRICT` is in the settable subset. XNU's own test `test_sysctl_kern_procargs_25397314` does exactly that and asserts that the env var disappears from another process's view. [S3][S18]

**`delete process.env.X`.** Node's delete removes the entry from libc's `environ` array. The strings `execve` put on the stack stay where they are, and `KERN_PROCARGS2` reads that stack. Probed: a Node process started with `FAKE_SECRET=…` that ran `delete process.env.FAKE_SECRET` saw `undefined` itself, while `ps eww -p <pid>` still showed `FAKE_SECRET=…`.

**SIP and hardened runtime limits.**
- SIP (on by default) is what makes the `CS_RESTRICT` stripping take effect. It doesn't protect ordinary third-party processes.
- Memory beyond the initial block, such as a key fetched later and held in Node's heap or the CLI's heap, needs a task port. `task_for_pid` failed (`KERN_FAILURE`) from a normal same-uid process, both inside and outside srt. It needs root or the debugger entitlement, and the hardened runtime refuses even those unless the target has `get-task-allow`. srt's profile also allows `mach-priv-task-port` only `(target same-sandbox)`. [S4]

## 2. macOS: srt's Seatbelt profile and Claude Code's Bash sandbox

srt's generated profile (`macos-sandbox-utils.ts`, starting at L1081) [S4]:

```
(version 1)
(deny default ...)
(allow process-exec)
(allow process-fork)
(allow process-info* (target same-sandbox))
(allow signal (target same-sandbox))
(allow mach-priv-task-port (target same-sandbox))
...
(allow sysctl-read (sysctl-name "hw.…") … (sysctl-name "kern.argmax") …
  (sysctl-name-prefix "kern.proc.all") (sysctl-name-prefix "kern.proc.pgrp.")
  (sysctl-name-prefix "kern.proc.pid.") …)
```

`kern.procargs2` isn't listed, and the `sysctl-read` filter does apply to numeric MIBs: XNU builds the canonical name string and passes it to `mac_system_check_sysctlbyname` [S19]. Probed under srt 0.0.83 with an empty-allow config:

| From inside srt | Result |
| :- | :- |
| `sysctl(CTL_KERN, KERN_BOOTTIME)` (not allowlisted) | `EPERM`, as the profile intends |
| `sysctl(CTL_KERN, KERN_PROCARGS2, <unsandboxed node pid>)` | **succeeds, environment included**, including the var Node deleted |
| `KERN_PROCARGS2` on an unsandboxed `claude -p` 2.1.274 | **succeeds, environment included** |
| `KERN_PROCARGS2` on an Apple binary (`/bin/sleep`) | succeeds, env stripped (`CS_RESTRICT`) |
| `/bin/ps eww` | `Operation not permitted`, because `ps` is setuid root and the sandbox refuses the exec. That isn't a defence: a 15-line ctypes script calls the sysctl directly |
| `sysctl kern.procargs2` (CLI) | `EPERM` on its name→OID lookup, not on procargs itself |
| `proc_pidpath(other)`, `proc_listpids(ALL)` | succeed, despite `process-info* (target same-sandbox)` |
| `task_for_pid(other)` | fails (`KERN_FAILURE`), same as unsandboxed |

Why procargs gets through isn't answerable from public source: Sandbox.kext is closed. The result is empirical, from this machine, 2026-11-22.

**Claude Code's Bash sandbox** is built on the same package [S6]. The 2.1.274 binary contains the same rule strings: `process-info* (target same-sandbox)` and the same `sysctl-name-prefix` list, with no procargs entry. So expect the same exposure for the agent's own shell. The docs list the environment as "Inherited from Claude Code, including any secrets in its environment". The fixes they name, `sandbox.credentials` `deny` (which unsets vars before each sandboxed command) and `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`, strip only what the child inherits. [S5][S6]

**Answer.** On macOS, a sandboxed command can read the **initial** environment of: sous-chef's Node process; the Claude Code CLI of any Run, including its `ANTHROPIC_API_KEY` if that came from the SDK's `env`; and an unsandboxed git child of any Run while it runs, including `SOUS_CHEF_PUSH_TOKEN` if that is in its environment. It can't read process memory beyond that.

## 3. Linux: bwrap, /proc and ptrace

srt's `wrapCommandWithSandboxLinux` [S7]:

- always starts with `['--new-session', '--die-with-parent']` (L3611).
- always adds `--unshare-pid` (L3810). The comment says "If we don't have --unshare-pid, it is possible to escape the sandbox".
- always adds `--unshare-user` and `--cap-drop ALL` (L3823-3830).
- mounts a fresh `--proc /proc` unless `enableWeakerNestedSandbox` is set, in which case it uses `--bind /proc /proc` (L3832-3842). The comment says "If we don't have --proc, it is possible to read host /proc and leak information about code running outside the sandbox".
- uses `--unshare-net` when network is restricted.

The source states the consequence itself: "Every sandbox this library starts has its own PID namespace and a fresh /proc, so none of them can reach /proc/<this pid> either." Each wrapped command is its own bwrap, so two Runs' sandboxes can't see each other's processes. Claude Code's docs say the same for the agent's shell: `enableWeakerNestedSandbox` "exposes process information to sandboxed commands". The env-scrub section adds that with `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` "the scrub also runs Bash subprocesses in an isolated PID namespace so they can't read host process environments through `/proc`". [S6][S5]

**ptrace.** Inside the sandbox, other processes aren't visible, and the sandbox runs in a child user namespace with all capabilities dropped. It can't hold `CAP_SYS_PTRACE` in the host namespace, so it can't ptrace host processes. ptrace matters only for **unsandboxed** same-uid code (for example a hook, an `excludedCommands` entry, or anything else the developer runs):

- `/proc/<pid>/environ` needs `PTRACE_MODE_READ_FSCREDS`: same uid, and the target is dumpable. It shows only the initial environment. [S8][S9]
- `/proc/<pid>/fd/*` needs `PTRACE_MODE_READ_FSCREDS`. Opening `/proc/<pid>/fd/N` for a pipe opens the same pipe, so a same-uid reader could race the helper for its contents. [S8]
- `/proc/<pid>/mem` needs `PTRACE_MODE_ATTACH_FSCREDS`. [S8]
- Yama `ptrace_scope=1` adds restrictions only when `mode & PTRACE_MODE_ATTACH`, so it blocks `mem` and `PTRACE_ATTACH` on non-descendants but not `environ` or `fd`. [S10]
- `prctl(PR_SET_DUMPABLE, 0)` makes all three fail for non-`CAP_SYS_PTRACE` callers. [S9]

**One gap on Linux to probe:** Claude Code's Read and Grep tools run outside the sandbox. With `permissions.blockReadsOutsideWorkingDirectories` they should refuse `/proc/<pid>/environ` (ADR 0002 already sets this switch). A probe should confirm it.

## 4. Claude Code: apiKeyHelper, fd handoff, env scrub, config dir

**`apiKeyHelper`** [S11][S12]:

- Configuration: a settings key, scope "Any file", whose value is a shell command line. It can be set through the SDK `settings` option, the flag-settings layer that becomes `--settings` and therefore argv. `settingSources: []` doesn't drop the flag layer.
- When it runs: "through the system shell, `/bin/sh` on macOS and Linux", and the output is sent as both `X-Api-Key` and `Authorization: Bearer`. The value is cached and re-run after 5 minutes (`CLAUDE_CODE_API_KEY_HELPER_TTL_MS`), on a 401 or 403, and before a request when a cached JWT has expired.
- Where it runs: outside the sandbox. The docs say helper commands "such as … `apiKeyHelper` run with your full access".
- Probed: the helper got Claude Code's environment, which held no key. The `SessionStart` hook's environment didn't contain the helper's output.
- The key lives only in the CLI's heap, never in its initial environment.
- Precedence: `ANTHROPIC_AUTH_TOKEN` > `ANTHROPIC_API_KEY` > `apiKeyHelper`, so neither variable may be set. [S12]
- The helper re-runs every 5 minutes, so its source has to persist for the whole Run: a 0600 file outside every sandbox's `allowRead`, or a sous-chef unix socket. srt blocks `AF_UNIX` by default, through Seatbelt on macOS and seccomp on Linux.

**`CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`** is undocumented, but 2.1.274 reads it at startup next to the documented `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`. From the bundled code: the CLI reads `/dev/fd/<n>`, or the inherited descriptor up to a newline or EOF, caches the value, and then runs `delete process.env[...]`. Persisting the value to `/home/claude/.claude/remote/.api_key` happens only when `CLAUDE_CODE_REMOTE` is set. [S13]

Probed with `3<&0` from a pipe and `ANTHROPIC_BASE_URL` pointed at a local header logger:

- every request carried `x-api-key: sk-ant-api03-FDFAKE…`.
- the CLI's `KERN_PROCARGS2` showed `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR=3` and not the key.
- the hook's environment had neither the variable nor the key, and the hook's `/dev/fd` had no inherited fd 3.
- nothing key-related was written to `CLAUDE_CONFIG_DIR`.

It is undocumented, so pin the version and probe it.

**`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`** is documented. It "strip[s] credentials from the environments of the subprocesses Claude Code starts, such as Bash commands, hooks, and stdio MCP servers", matching by name or value shape. [S5]

| Variable | What the scrub does |
| :- | :- |
| `ANTHROPIC_API_KEY`, `AWS_SECRET_ACCESS_KEY` | Removes it |
| Names or values that look like credentials (`*_TOKEN`, `*_PASSWORD`, a `DATABASE_URL` with a password) | Removes it |
| `CLAUDE_CONFIG_DIR` | Removes it |
| Registry URLs (`PIP_INDEX_URL`, `NPM_CONFIG_REGISTRY`) | Keeps the URL and cuts the username and password from it |
| `GITHUB_TOKEN`, `GH_TOKEN`, `GH_ENTERPRISE_TOKEN`, `GITHUB_ENTERPRISE_TOKEN` | Leaves it in place |
| `HTTP(S)_PROXY` | Leaves it in place |
| `GIT_CONFIG_*` | Leaves it in place |

Side effects of the scrub: on Linux it puts every Bash command in its own PID namespace, it makes every command run sandboxed, and it turns auto-allow off. Before v2.1.251 it removed only the two named keys. **Its own text says "The Claude Code process keeps the credentials for its own API calls"**, so on macOS the initial environment stays readable.

**Config dir.** Probed with a fresh `CLAUDE_CONFIG_DIR` and `HOME`, `ANTHROPIC_API_KEY=<fake>`, and `claude -p`. Afterwards `.claude.json` held only `firstStartTime`, `machineID`, `userID`, migration flags and `seenNotifications`. There was no `customApiKeyResponses` and no `primaryApiKey`, and grep found no fragment of the key in any file, transcript included. From the bundled code:

- `customApiKeyResponses.approved` stores `key.trim().slice(-20)`, the last 20 characters. It is written only when a key is approved interactively or saved by a Console `/login`. That path also stores `primaryApiKey`, in the macOS Keychain or in the config file.
- In `-p` mode an env key is used without approval. [S13]
- `.credentials.json` is written by `/login`, under `CLAUDE_CONFIG_DIR` when that is set. [S12]

So "a fresh `CLAUDE_CONFIG_DIR` … holding only the API key" in ADR 0002's isolation amendment isn't needed. Claude Code doesn't put the key there itself in headless mode, and sous-chef shouldn't either.

## 5. git: credentials without environment

- **Inline helper reading an inherited fd works.** gitcredentials(7): a helper string starting with `!` is a shell snippet with the operation appended. An empty `credential.helper` resets the list. git exchanges credentials with helpers over stdin and stdout (the `key=value` protocol). [S14]
- **Probed with git 2.52** (Xcode build): `credential.helper=` plus `credential.http://127.0.0.1:<port>.helper=!f() { test "$1" = get || exit 0; echo username=x-access-token; printf password=; head -n1 <&3; }; f`, with the token on fd 3 from a pipe. The helper ran as `get`, `/dev/fd` showed 0–7 (fd 3 inherited through `git` → `git` → `git-remote-http` → `sh`), and the server received `x-access-token:<token>`. While the request was in flight, `KERN_PROCARGS2` on `git`, the re-exec and `git-remote-http` didn't contain the token.
- **Does git close extra fds?** No. `run-command.c` has no `closefrom` or `close_range`. It only sets `FD_CLOEXEC` on fds it opens itself (`set_cloexec`, `/dev/null` opened `O_CLOEXEC`). An fd the parent passes without CLOEXEC survives every exec. Node's `spawn` with `stdio: [..., ..., ..., 'pipe']` gives the child fd 3 without CLOEXEC. [S15]
- **The pipe is read once.** git also calls the helper for `store` or `erase` after the request. The `test "$1" = get || exit 0` guard handles that, and a second `get` (for example a retry after 401) sees EOF. So write the token followed by a newline and close the write end.
- **Argv and env of git's own children.** The password is sent to the helper on stdin and read back on stdout, and `git-remote-http` asks for it itself (`credential_fill` in the remote-curl process). It doesn't appear in argv or env (probed above). The one trap: `git -c k=v` exports every `-c` pair to children as `GIT_CONFIG_PARAMETERS` (`config.c` `git_config_push_parameter` → `setenv(CONFIG_DATA_ENVIRONMENT)`) [S16], on top of being in git's own argv. So never put the token literally in a `-c` value or a helper string.
- **`GIT_ASKPASS`** is consulted only when no helper supplies the credential. The program gets the prompt as argv and prints the answer on stdout [S14]. It works too, and the variable holds only a path, but it is asked separately for username and password and has no `get`/`store` distinction. The fd helper is simpler.
- **A unix-socket helper** (like `git-credential-cache`) works as well. Inside srt, sandboxed processes can't reach the socket unless it is in `allowUnixSockets`. Unsandboxed same-uid code can, unless the socket checks the peer or uses a one-time secret.
- **Can a same-uid process read a pipe another process holds?**
  - macOS: `lsof` lists it but can't read it. There is no `/proc`, and getting someone else's fd needs their task port (fileport), which fails as shown in §1.
  - Linux: an unsandboxed same-uid process with `PTRACE_MODE_READ` access can `open("/proc/<pid>/fd/3")` and consume the bytes [S8]. A sandboxed one can't see the pid.
  - Both: set `PR_SET_DUMPABLE=0` on Linux, keep the pipe's lifetime short, and pass the token through the pipe only to the push git.

## 6. Agent SDK `env`

`sdk.d.ts` (0.3.330): `env` "REPLACES the subprocess environment entirely … When omitted, the subprocess inherits `process.env`." [S17] In `sdk.mjs`:

- the transport destructures `env:c={...process.env}` and builds `{command, args, cwd, env: c, signal}`.
- `spawnLocalProcess` calls `child_process.spawn(t, n, {cwd, stdio:["pipe","pipe","pipe"], signal, env: o})`. So `options.env` is the CLI's initial environment, readable on macOS through `ps eww` and `KERN_PROCARGS2`, and on Linux through `/proc/<pid>/environ` from outside a sandbox.
- `settings` becomes `--settings` (argv).
- `spawnClaudeCodeProcess(options)` replaces the spawn and receives the same `{command, args, cwd, env, signal}`. That is the hook for adding `stdio[3]` to use `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`.

## Implications for issue #24 (not decisions)

- On macOS, **no credential may be in the initial environment of any long-lived process** while untrusted code runs as the same uid. That covers sous-chef itself (read `gh auth token` or the key at runtime, not from `env`), the Claude Code CLI (use the key fd or `apiKeyHelper`, never `env`), and the push git (use the fd helper, never env).
- If a credential does arrive in sous-chef's own launch env (the developer exports `SOUS_CHEF_PUSH_TOKEN`), deleting it from `process.env` doesn't help. Two things do: re-exec without it, or set `CS_RESTRICT` on self through `csops`, which needs a native call. ADR 0002's planned key-discovery probe should check initial environments, not just inherited ones.
- Probe cases that fall out:
  - from inside the core srt sandbox and from the agent's Bash, `KERN_PROCARGS2` (macOS) or `/proc/<pid>/environ` (Linux) on sous-chef's pid, the CLI's pid, and a concurrently running push git, asserting that no held credential appears (raw, base64, URL-encoded).
  - the Read tool on `/proc/self/environ` and `/proc/<sous-chef pid>/environ` on Linux.
  - the CLI's `.claude.json` and transcript containing no key fragment, including the last 20 characters.

## Sources

- [S1] macOS `ps(1)`, `-E` and `-e` (local `man ps`, macOS 27.2)
- [S2] XNU `bsd/kern/kern_sysctl.c`, `sysctl_procargsx()`: https://github.com/apple-oss-distributions/xnu/blob/main/bsd/kern/kern_sysctl.c
- [S3] XNU `bsd/kern/kern_cs.c` `cs_restricted()`, and `bsd/kern/kern_proc.c` `csops` `CS_OPS_SET_STATUS` settable flags: https://github.com/apple-oss-distributions/xnu/blob/main/bsd/kern/kern_cs.c, https://github.com/apple-oss-distributions/xnu/blob/main/bsd/kern/kern_proc.c
- [S4] srt macOS Seatbelt profile generator: https://github.com/anthropics/sandbox-runtime/blob/0bbebed0aa9a8e628dc4df8bc30293550a1988eb/src/sandbox/macos-sandbox-utils.ts#L1081-L1225, plus the experiments in this note
- [S5] Claude Code env vars, `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` and "What the subprocess environment scrub removes": https://code.claude.com/docs/en/env-vars
- [S6] Claude Code sandboxing, "What the sandbox restricts", "What runs outside the sandbox", `sandbox.credentials`, `enableWeakerNestedSandbox`: https://code.claude.com/docs/en/sandboxing
- [S7] srt Linux bwrap args: https://github.com/anthropics/sandbox-runtime/blob/0bbebed0aa9a8e628dc4df8bc30293550a1988eb/src/sandbox/linux-sandbox-utils.ts#L3611 and #L3803-L3843, comment at #L884-L893
- [S8] `proc_pid_environ(5)`, `proc_pid_mem(5)`, `proc_pid_fd(5)`: https://man7.org/linux/man-pages/man5/proc_pid_environ.5.html, https://man7.org/linux/man-pages/man5/proc_pid_mem.5.html, https://man7.org/linux/man-pages/man5/proc_pid_fd.5.html
- [S9] `ptrace(2)`, "Ptrace access mode checking": https://man7.org/linux/man-pages/man2/ptrace.2.html
- [S10] Yama: https://docs.kernel.org/admin-guide/LSM/Yama.html, and `security/yama/yama_lsm.c` (`if (mode & PTRACE_MODE_ATTACH)`): https://github.com/torvalds/linux/blob/master/security/yama/yama_lsm.c
- [S11] Settings reference, `apiKeyHelper`: https://code.claude.com/docs/en/settings-reference#apikeyhelper
- [S12] Authentication, credential management and precedence: https://code.claude.com/docs/en/authentication
- [S13] Claude Code 2.1.274 native binary (`/opt/homebrew/Caskroom/claude-code/2.1.274/claude`), bundled JS: `nq()` → `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR`, `K8(e){return e.trim().slice(-20)}`, the `customApiKeyResponses` writer. Undocumented, so it may change
- [S14] gitcredentials(7) (v2.52.0), "Requesting credentials" and "Custom helpers": https://git-scm.com/docs/gitcredentials, and git-credential(1): https://git-scm.com/docs/git-credential
- [S15] git `run-command.c` (v2.52.0): https://github.com/git/git/blob/v2.52.0/run-command.c
- [S16] git `config.c` (v2.52.0), `git_config_push_parameter`: https://github.com/git/git/blob/v2.52.0/config.c
- [S17] `@anthropic-ai/claude-agent-sdk` 0.3.330 `sdk.d.ts` (`env`, `settings`, `spawnClaudeCodeProcess`) and `sdk.mjs` (`spawnLocalProcess`); Agent SDK TypeScript reference: https://code.claude.com/docs/en/agent-sdk/typescript
- [S18] XNU test `tests/test_sysctl_kern_procargs_25397314.m`: https://github.com/apple-oss-distributions/xnu/blob/main/tests/test_sysctl_kern_procargs_25397314.m
- [S19] XNU `bsd/kern/kern_newsysctl.c`, `sysctl_root()` → `mac_system_check_sysctlbyname(…, namestring, …)`: https://github.com/apple-oss-distributions/xnu/blob/main/bsd/kern/kern_newsysctl.c
