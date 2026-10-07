---
type: failure
concepts: [shell-execution, process-tree-kill]
harnesses: [pi, codex]
---
**Symptom** — Four faces of one problem:
- Aborting `sleep 4 && echo hello` left it running ("echo hello" appeared after abort).
- On Windows, commands that spawn daemonized descendants (agent-browser) made the bash tool **spin forever**.
- On POSIX, late output from descendants still writing after the shell exited was **truncated**.
- Detached children survived pi's exit; bash-executor temp streams leaked fds.

**Root cause** — "Process exited", "output finished" and "pipes closed" are three different events. `exec()` killed only the parent shell; descendants inherit stdout/stderr so `close` never fires while they live; a fixed post-exit deadline destroyed streams mid-write; detached process groups outlive the parent.

**Fix · [[pi]]**
- `6e9fa8dde` 2025-11-11 — spawn `detached` (own process group) and `process.kill(-pid, SIGKILL)` the whole tree on abort (`packages/coding-agent/src/utils/shell.ts:187-218`).
- `25b185f39` 2026-03-19 (#2389) — `waitForChildProcess` resolves on `exit` + pipe `end` or an idle grace, not only `close` (`packages/coding-agent/src/utils/child-process.ts:49-137`).
- `9b7948c4c` 2026-04-16 — track detached child pids; `killTrackedDetachedChildren()` on SIGHUP/SIGTERM/exit in interactive/print/rpc modes (`shell.ts:167-182`; `packages/coding-agent/src/modes/print-mode.ts:58`, `packages/coding-agent/src/modes/rpc/rpc-mode.ts:374`).
- `e4f847ff6` 2026-04-27 (#3786) — close bash-executor temp streams.
- `3fa409562` 2026-06-15 (#5303/#5753) — `EXIT_STDIO_GRACE_MS = 100` idle timer **re-armed on every data chunk**, so a descendant still writing is not cut off while a quiet inherited handle still releases (`child-process.ts:16`).

**Fix · [[codex]]**
- Symptom in codex: "many reports of codex hanging when calling certain tools" (issue #3204) — the output reader waited for stdout/stderr EOF after the child exited, but grandchildren inheriting the pipes kept them open forever.
- `73ed30d7e5` 2025-11-12 (#6575) — bounded drain after exit `IO_DRAIN_TIMEOUT_MS = 2_000` ("2 s should be plenty for local pipes") in the legacy exec path (`codex-rs/core/src/exec.rs:94`); cancellation grace 50 ms (`exec.rs:71`).
- Unified exec: after exit waits at most `POST_EXIT_CLOSE_WAIT_CAP = 50 ms` for pipe close (`codex-rs/core/src/unified_exec/process_manager.rs:1573-1640`); `c2ca51273f` 2026-02-09 exit notify instead of a fixed grace; trailing-output grace 100 ms (`codex-rs/core/src/unified_exec/async_watcher.rs:34`).
- `82c981cafc` 2026-02-06 process-group cleanup for stdio MCP servers ("prevent orphan process storms"); `779e9114ae` 2026-08-13 "Reap orphaned processes in Linux sandboxes".
- Deliberate contrast: interrupts do **not** kill long-lived unified-exec processes (`ba463a9dc7`) — see [[interrupt-kills-background-processes]].

**Lesson** — Track exit, pipe drain and descendant lifetime separately: bound the post-exit drain, kill the process group, and clean up detached children (and orphaned server/sandbox processes) at shutdown.

Related: [[shell-execution]] · [[process-tree-kill]] · [[windows-process-tree-and-shells]] · [[pi--shell-execution|pi]] · [[no-background-bash]] · [[codex--shell-execution|codex]] · [[mcp-integration]]
