---
type: implementation
harness: codex
concept: process-tree-kill
commit: 622e9e3696
files: [codex-rs/utils/pty/src/process_group.rs:1, codex-rs/utils/pty/src/process_group.rs:92, codex-rs/utils/pty/src/process_group.rs:269, codex-rs/core/src/spawn.rs:101, codex-rs/core/src/exec.rs:63, codex-rs/core/src/exec.rs:1025, codex-rs/utils/pty/src/win/job.rs:44, codex-rs/core/src/context/turn_aborted.rs:10]
---
[[process-tree-kill]] in [[codex]].

## Mechanism
- Helpers `codex-rs/utils/pty/src/process_group.rs:1-18`: `set_process_group` in `pre_exec` (child = own group); `detach_from_tty` = `setsid()` (falls back to `setpgid` on `EPERM`) so non-interactive children have no controlling TTY (`:50-60`); `kill_process_group_by_pid` = `getpgid` + `killpg(SIGKILL)`, ESRCH ignored (`:92-115`); `set_parent_death_signal` (Linux) `PR_SET_PDEATHSIG=SIGTERM` + re-check `getppid` to close the fork/exec race (`:24-41`); macOS group signals retry against individual members when the group signal is denied (`:16-17`, `:274-284`); non-Unix: no-ops.
- Shell tool spawn: `StdioPolicy::RedirectForShellTool` → `detach_from_tty` + parent-death signal (`codex-rs/core/src/spawn.rs:101-114`).
- Exec lifecycle (`codex-rs/core/src/exec.rs:1025-1080`): timeout → `kill_child_process_group` + `start_kill`, synthetic exit `128 + 64`; cancellation → `terminate_process_group` (TERM), wait `CANCELLATION_TERMINATION_GRACE_PERIOD = 50 ms`, then SIGKILL the group; Ctrl-C → SIGKILL group. Output drain after exit bounded by `IO_DRAIN_TIMEOUT_MS = 2_000`; drain failure kills the group (`:1145-1156`) — fix for grandchildren holding pipes ([[bash-descendants-hang-or-lose-output]]).
- Windows: Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE | BREAKAWAY_OK`, or without breakaway (`codex-rs/utils/pty/src/win/job.rs:44-66`); app-server daemon same (`codex-rs/app-server-daemon/src/backend/windows.rs:300`); MXC assigns kill-on-close job before resuming the suspended child and kills descendants on foreground exit (`codex-rs/mxc-sandbox/README.md:19-21`, `:75-79`).
- Other groups: stdio MCP servers (`82c981cafc`; macOS per-process fallback `f2d825533c`), exec-server / code-mode / git / rmcp children `process_group(0)` (e.g. `codex-rs/exec-server/src/client_transport.rs:851`, `codex-rs/git-utils/src/git_process.rs:35`); orphan reaping inside Linux sandboxes (`779e9114ae`); Linux sandbox proxy helper `PR_SET_PDEATHSIG` (`codex-rs/linux-sandbox/src/proxy_lifecycle.rs:123`).
- **User interrupt does not kill background unified-exec processes**: the abort marker tells the model "Any running unified exec processes may still be running in the background. If any tools/commands were aborted, they may have partially executed." (`codex-rs/core/src/context/turn_aborted.rs:10`). See [[interrupt-kills-background-processes]].

## Constants
| name | value | path:line |
|---|---|---|
| cancel TERM→KILL grace | `CANCELLATION_TERMINATION_GRACE_PERIOD` = 50 ms | `codex-rs/core/src/exec.rs:71` |
| post-exit IO drain | `IO_DRAIN_TIMEOUT_MS` = 2_000 | `codex-rs/core/src/exec.rs:94` |
| default exec timeout | `DEFAULT_EXEC_COMMAND_TIMEOUT_MS` = 10_000 | `codex-rs/core/src/exec.rs:63` |
| timeout exit code | 124 (synthetic signal 64) | `codex-rs/core/src/exec.rs:68-70` |

## Evolution
- 2025-07-22 `d51654822f` `PR_SET_PDEATHSIG` "to ensure child processes are killed in a timely manner".
- 2025-11-07 `a2fdfce02a` "Kill shell tool process groups on timeout (#5258)".
- 2025-11-12 `73ed30d7e5` avoid hang when grandchild shares stdout/stderr.
- 2026-01-14 `577e1fd1b2` piped process alternative to PTY with group kill.
- 2026-02-06 `82c981cafc` process-group cleanup for stdio MCP servers ("orphan process storms").
- 2026-05-27 `9152ebd289` linux-sandbox: preserve shell cleanup on interruption.
- 2026-08-05 `f2d825533c` macOS per-process MCP cleanup; 2026-08-13 `779e9114ae` reap orphans in Linux sandboxes; 2026-09-09 `808b3411fd` harden capture cleanup; 2026-09-18 `d6fb836f31` macOS member fallback in shared helpers.

## Versus pi
- [[pi--process-tree-kill]]: `kill(-pid, SIGKILL)` with no grace for bash, `taskkill /F /T` on Windows, tracked-pid sweep on shutdown, remote dead-man switch. Codex: TERM → 50 ms → KILL on cancel, parent-death signal, Job Objects on Windows, but deliberately leaves background unified-exec sessions alive on interrupt.
