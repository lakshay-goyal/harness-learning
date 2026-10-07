---
type: concept
stage: tools
tier: candidate
aliases: [killProcessTree, detached, setsid, process group, "kill(-pgid)", taskkill /T, KILL_ON_JOB_CLOSE, killTrackedDetachedChildren, SILENCE_LIMIT_MS, kill_all, dead-man switch, process-group-kill, dead-man-switch-process-cleanup, forceKillAfter, killTree, OPENCODE_PID]
harnesses: [pi, opencode]
---
Run every agent-launched command in its own process group / job object and kill the whole tree (not just the shell pid) on abort, timeout, harness shutdown, or — for remote executors — when the controlling client goes silent (dead-man switch).

## Why
- Killing only the shell leaves `sleep 4 && echo hello`, dev servers, test runners running after the user hit Esc ([[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]]).
- Agents exit or lose connectivity mid-command; without tracking + group kill, orphans accumulate on the host or remote box.
- Platform asymmetry: Unix process groups vs Windows job objects / `taskkill /T`; using Unix tricks on Windows breaks console stdio (pi `24dec9fcd`).

## Design space
- **Kill pid only** (Node `exec` default) — pi pre-`6e9fa8dde`, rejected.
- **New process group + `kill(-pgid, SIGKILL)`** on Unix — pi's choice; no SIGTERM grace for the bash tool (extension `exec` helper does SIGTERM → 5 s → SIGKILL).
- **Windows**: `taskkill /F /T` by absolute System32 path (pi) vs Job Object `KILL_ON_JOB_CLOSE` (pi-env daemon).
- **Shutdown sweep**: track live child pids and kill on SIGHUP/SIGTERM/exit (pi `9b7948c4c`) vs leave orphans.
- **Normal completion**: kill remaining group members when the shell exits (reaps `cmd &`) vs leave them — pi leaves them (inferred).
- **Remote dead-man switch**: daemon kills all groups it started on heartbeat silence or stdin EOF (pi-env, 30 s) vs rely on SSH session teardown.
- **Liveness signal**: count frames vs any received bytes (pi-env counts bytes so slow large frames aren't "silence").
- **Grace period**: TERM → 3 s → KILL for tool shells, TERM → 200 ms → KILL in the core helper (opencode).
- **Spawned servers**: walk stdio MCP server descendants with `pgrep -P` and SIGTERM them on shutdown (opencode).

## Implementations
- [[pi--process-tree-kill|pi]] — bash spawned `detached` (Unix), `killProcessTree` = `kill(-pid,SIGKILL)` / `taskkill /F /T`; tracked-pid sweep on shutdown; pi-env Rust daemon `setsid` / Job Object + 30 s silence dead-man switch.
- [[opencode--process-tree-kill|opencode]] — shell spawned `detached` with `stdin: "ignore"`, `kill({forceKillAfter: "3 seconds"})` on abort/timeout; core `killTree` = `kill(-pid)` / `taskkill /t`; MCP descendants SIGTERMed on exit.

## Failures
- [[mcp-child-processes-orphaned]]
- [[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]]
- [[liveness-timeout-counts-frames-not-bytes]]

## Tradeoffs
- [[bash-timeout-default-vs-none]]

## Related
[[shell-execution]] · [[abort-propagation]] · [[remote-execution-env]] · [[tool-only-isolation]] · [[no-background-bash]] · [[no-bash-default-timeout]] · [[mcp-integration]]
