---
type: concept
stage: tools
tier: candidate
aliases: [killProcessTree, detached, setsid, process group, "kill(-pgid)", taskkill /T, KILL_ON_JOB_CLOSE, killTrackedDetachedChildren, SILENCE_LIMIT_MS, kill_all, dead-man switch, process-group-kill, dead-man-switch-process-cleanup, kill_process_group, kill_child_process_group, detach_from_tty, PR_SET_PDEATHSIG, CANCELLATION_TERMINATION_GRACE_PERIOD]
harnesses: [pi, codex]
---
Run every agent-launched command in its own process group / job object and kill the whole tree (not just the shell pid) on abort, timeout, harness shutdown, or — for remote executors — when the controlling client goes silent (dead-man switch).

## Why
- Killing only the shell leaves `sleep 4 && echo hello`, dev servers, test runners running after the user hit Esc ([[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]]).
- Agents exit or lose connectivity mid-command; without tracking + group kill, orphans accumulate on the host or remote box.
- Platform asymmetry: Unix process groups vs Windows job objects / `taskkill /T`; using Unix tricks on Windows breaks console stdio (pi `24dec9fcd`).

## Design space
- **Kill pid only** (Node `exec` default) — pi pre-`6e9fa8dde`, rejected.
- **New process group + `kill(-pgid, SIGKILL)`** on Unix — ✔ pi (no SIGTERM grace for the bash tool; extension `exec` helper does SIGTERM → 5 s → SIGKILL); ✔ codex (`setsid`/own group, `killpg(SIGKILL)` on timeout, TERM → 50 ms → KILL on cancel; macOS falls back to per-member signals when group signal is denied).
- **Windows**: `taskkill /F /T` by absolute System32 path (✔ pi) vs Job Object `KILL_ON_JOB_CLOSE` (pi-env daemon; ✔ codex pty/pipe children, app-server daemon, MXC job assigned before resume).
- **Parent-death signal**: child gets SIGTERM when the harness dies (`PR_SET_PDEATHSIG`, Linux) — ✔ codex, pid race re-checked; pi uses shutdown sweep instead.
- **Drain bound after exit**: kill the group if pipes stay open past a deadline (✔ codex `IO_DRAIN_TIMEOUT_MS` 2 s).
- **Shutdown sweep**: track live child pids and kill on SIGHUP/SIGTERM/exit (pi `9b7948c4c`) vs leave orphans.
- **Normal completion**: kill remaining group members when the shell exits (reaps `cmd &`) vs leave them — pi leaves them (inferred); codex Windows backends preserve descendants, MXC kills them.
- **User interrupt**: kill everything (pi) vs leave background sessions running and tell the model (✔ codex unified exec: "Any running unified exec processes may still be running in the background").
- **Remote dead-man switch**: daemon kills all groups it started on heartbeat silence or stdin EOF (pi-env, 30 s) vs rely on SSH session teardown.
- **Liveness signal**: count frames vs any received bytes (pi-env counts bytes so slow large frames aren't "silence").

## Implementations
- [[pi--process-tree-kill|pi]] — bash spawned `detached` (Unix), `killProcessTree` = `kill(-pid,SIGKILL)` / `taskkill /F /T`; tracked-pid sweep on shutdown; pi-env Rust daemon `setsid` / Job Object + 30 s silence dead-man switch.
- [[codex--process-tree-kill|codex]] — `codex-rs/utils/pty/src/process_group.rs` helpers (`setsid`, `killpg`, `PR_SET_PDEATHSIG`); TERM → 50 ms → KILL on cancel; Job Objects on Windows; background unified-exec sessions survive interrupt.

## Failures
- [[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]]
- [[liveness-timeout-counts-frames-not-bytes]]
- [[interrupt-kills-background-processes]] (codex) · [[provider-stream-ignores-abort]]

## Related
[[os-level-sandbox]] · [[shell-execution]] · [[abort-propagation]] · [[remote-execution-env]] · [[tool-only-isolation]] · [[no-background-bash]] · [[no-bash-default-timeout]]
