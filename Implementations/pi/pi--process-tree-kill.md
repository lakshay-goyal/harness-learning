---
type: implementation
harness: pi
concept: process-tree-kill
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/bash.ts:114, packages/coding-agent/src/utils/shell.ts:167, packages/coding-agent/src/utils/shell.ts:187, packages/coding-agent/src/core/exec.ts:52, packages/durable/src/env/node.ts:274, packages/env/daemon/src/main.rs:33, packages/env/daemon/src/main.rs:464, packages/env/daemon/src/sys/unix.rs:286]
---
[[process-tree-kill]] in [[pi]].

## Mechanism
- **Bash tool spawn** (`packages/coding-agent/src/core/tools/bash.ts:111-118`): `detached: process.platform !== "win32"` → new process group on Unix (`:114`); `stdio[0]` ignored unless command goes via stdin (legacy WSL); pid tracked `trackDetachedChildPid` (`:123`).
- **Abort**: `signal` listener → `killProcessTree(child.pid)` (`:126-128,142-145`); **timeout** (only if model passed one — [[no-bash-default-timeout]]) → same kill (`:131-136`); inner op throws `"aborted"` / `"timeout:N"` (`:148-153`); tool wrapper rethrows as output + `"Command aborted"` / `"Command timed out after N seconds"` (`bash.ts:375,379`).

- **`killProcessTree`** (`packages/coding-agent/src/utils/shell.ts:187-218`): Unix `process.kill(-pid, "SIGKILL")` (whole group), fallback single pid; Windows `%SystemRoot%\System32\taskkill.exe /F /T /PID` by absolute path "so cleanup does not depend on PATH", detached, `error` swallowed. **No SIGTERM grace** for the bash tool.
- **Shutdown sweep**: module-level `trackedDetachedChildPids` Set (`shell.ts:167-182`); `killTrackedDetachedChildren()` on SIGHUP/SIGTERM/exit in interactive (`interactive-mode.ts:4314,4344,4381`), print (`print-mode.ts:58`) and RPC (`rpc-mode.ts:374`) modes (`9b7948c4c`).
- **Normal completion**: `finally` only untracks the pid (`bash.ts:159-160`); the group is not killed → backgrounded grandchildren (`cmd &`) outlive the tool call and are not swept at shutdown (inferred from code; untested).
- **Exit wait** (related, [[shell-execution]]): `waitForChildProcess` resolves on `exit` + pipe idle `EXIT_STDIO_GRACE_MS = 100`, re-armed per chunk, so detached descendants holding pipes can't hang the tool (`packages/coding-agent/src/utils/child-process.ts:16,49-137`; `25b185f39`, `3fa409562`).
- **Signal exit code**: signal-killed shell ⇒ `128 + signum` (`bash.ts:155-158`; `a8b3dd199`, #9577) so a kill never reads as success.
- **Extension helper** `pi.exec` (`packages/coding-agent/src/core/exec.ts:52-63`): `SIGTERM` then `SIGKILL` after 5 s, single process (`shell:false`), not group.
- **Durable `NodeExecutionEnv`**: POSIX children `detached`; `killProcessTree` = `kill(-pid,SIGKILL)` / `taskkill` (`packages/durable/src/env/node.ts:274-298,805`); `activeChildPids` for owner-shutdown `cleanup()` (`:610,1221-1224`). Contract: "Aborting the context or a timeout kills only this command's processes"; `cleanup()` = "Kill every command this environment still runs; for its owner's shutdown, never for one request" (`packages/durable/src/env/index.ts:309-321`).
- **pi-env daemon (remote)**:
  - Unix: child `setsid()`, default signal dispositions, empty mask (libuv parity) (`packages/env/daemon/src/sys/unix.rs:266-280`); `kill_tree` = `kill(-pgid, SIGKILL)` else pid (`:286-295`). Windows: Job Object `KILL_ON_JOB_CLOSE` (`sys/windows.rs:524-548`). `cleanup()` ⇒ exit 137 (`packages/env/docs/semantics.md:54-55`).
  - **Dead-man switch**: ping thread every `PING_INTERVAL` 5 s; if no bytes received for > `SILENCE_LIMIT_MS` 30 s → `kill_all(groups)` + `exit(0)` — "A client that went silent (a phone that lost its network) leaves nothing running." (`main.rs:33-34,464-474`). On stdin EOF / any input end → `kill_all` (`main.rs:483-488`).
  - Liveness counts **bytes**, not frames: `Seen<R>` reader stamps `last_seen` on every read (`main.rs:313-327`; `46d0ff936`).
  - Client mirrors: ping 5 s, teardown after 30 s silence (`packages/env/src/connection.ts:13-15,288-294`).
  - `cancel {mode:"kill"}` kills without aborting so the command settles with killed status (`packages/env/docs/protocol.md:45-50`).

## Constants
| name | value | path:line |
|---|---|---|
| bash kill signal | `SIGKILL` to `-pid` | packages/coding-agent/src/utils/shell.ts:208 |
| `EXIT_STDIO_GRACE_MS` | 100 ms | packages/coding-agent/src/utils/child-process.ts:16 |
| `pi.exec` SIGTERM→SIGKILL | 5000 ms | packages/coding-agent/src/core/exec.ts:57-61 |
| `PING_INTERVAL` | 5 s | packages/env/daemon/src/main.rs:33 |
| `SILENCE_LIMIT_MS` | 30 000 | packages/env/daemon/src/main.rs:34 |
| client start+hello timeout `START_TIMEOUT_MS` | 60 s | packages/env/src/connection.ts:16-17 |

## Evolution
- 2025-11-11 `6e9fa8dde` "kill entire process tree immediately": previous `exec()` killed only the parent shell (`sleep 4 && echo hello` survived abort) → spawn detached + `kill(-pgid)`.
- 2026-03-19 `25b185f39` Windows: don't wait on inherited handles of detached descendants (#2389).
- 2026-04-16 `9b7948c4c` track + kill detached bash children on shutdown.
- 2026-05-01 `24dec9fcd` remove `detached:true` on Windows — it broke `pwsh.exe` console stdio; `taskkill /T` needs no group (#4013).
- 2026-06-15 `3fa409562` idle grace re-armed per chunk (#5753).
- 2026-08-26 `7af2d27dc` absolute `System32\taskkill.exe` + async error swallow — aborts crashed when taskkill missing from PATH (#6596).
- 2026-09-17 `a8b3dd199` 128+signal exit codes (#9577).
- 2026-10-05 `ba03e03f2` pi-env daemon with group kill; `46d0ff936` `Seen` byte-liveness, worker pool, control/bulk queues.

## Evidence commits
`6e9fa8dde`, `9b7948c4c`, `24dec9fcd`, `7af2d27dc`, `25b185f39`, `3fa409562`, `a8b3dd199`, `ba03e03f2`, `46d0ff936`

## Quirks
- No graceful SIGTERM phase: tools that need cleanup (DB, git lock files) are SIGKILLed.
- `(cmd &)` / `nohup` children survive tool completion and pi exit (group not killed on success; pid untracked) — matches the "No background bash. Use tmux" stance ([[no-background-bash]]) but leaks silently (inferred).
- Remote dead-man switch is the only containment in pi-env (no path allowlist/chroot) — see [[tool-only-isolation]].

## Failures
- [[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]] · [[liveness-timeout-counts-frames-not-bytes]]
