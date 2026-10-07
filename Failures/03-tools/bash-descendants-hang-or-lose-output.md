---
type: failure
concepts: [shell-execution, process-tree-kill]
harnesses: [pi]
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

**Lesson** — Track exit, pipe drain and descendant lifetime separately: kill the process group, resolve on exit + idle pipes, and clean up detached children at shutdown.

Related: [[shell-execution]] · [[process-tree-kill]] · [[windows-process-tree-and-shells]] · [[pi--shell-execution|pi]] · [[no-background-bash]]
