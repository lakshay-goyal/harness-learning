---
type: implementation
harness: opencode
concept: process-tree-kill
commit: ecc4916b5a
files: [packages/opencode/src/tool/shell.ts:292-310, packages/opencode/src/tool/shell.ts:540-566, packages/core/src/shell.ts:12, packages/core/src/shell.ts:30-58, packages/opencode/src/mcp/index.ts:418-430, packages/opencode/src/mcp/index.ts:536-550, packages/opencode/src/index.ts:77]
---
[[process-tree-kill]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Shell tool spawn** (`packages/opencode/src/tool/shell.ts:292-310`): fresh child per call via Effect `ChildProcess`, `stdin: "ignore"`, `detached: process.platform !== "win32"` → own process group on POSIX. Windows PowerShell spawned with `-NonInteractive`, not detached.
- **Abort / timeout** (`packages/opencode/src/tool/shell.ts:540-566`): exit races `Effect.sleep(timeout + 100 ms)` and the abort signal; on either, `handle.kill({forceKillAfter: "3 seconds"})` (TERM, then KILL after 3 s). Timeout appends model-facing text: "…terminated command after exceeding timeout N ms. If this command is expected to take longer and is not waiting for interactive input, retry with a larger timeout value" (`packages/opencode/src/tool/shell.ts:564`).
- **Core helper** `killTree(proc)` (`packages/core/src/shell.ts:30-58`): Windows `taskkill /pid <pid> /f /t`; POSIX `process.kill(-pid, SIGTERM)` → 200 ms → `SIGKILL` to the group; falls back to the pid if group kill throws.
- **MCP stdio servers**: on shutdown, for each stdio client, walk descendants with repeated `pgrep -P <pid>` (`packages/opencode/src/mcp/index.ts:418-430`) and `SIGTERM` each before `client.close()` (`packages/opencode/src/mcp/index.ts:536-550`). The harness pid is exported as `OPENCODE_PID` (`packages/opencode/src/index.ts:77`), inherited by spawned servers (`c4c0b23bff`; purpose per commit title, use by servers unverified).
- **Normal completion**: no group kill when the shell exits normally; background `cmd &` survivors are left running (inferred; no reap path found).

### v2 runtime
- v2 bash tool spawns `detached` on POSIX with `stdin: "ignore"` and `forceKillAfter: Duration.seconds(3)` (`packages/core/src/tool/bash.ts:160-163`).

## Constants
| name | value | path:line |
|---|---|---|
| shell kill grace | `forceKillAfter: "3 seconds"` | `packages/opencode/src/tool/shell.ts:550,554`; `packages/core/src/tool/bash.ts:163` |
| core kill grace | `SIGKILL_TIMEOUT_MS = 200` | `packages/core/src/shell.ts:12` |
| default shell timeout | `bashDefaultTimeoutMs ?? 2 * 60 * 1000` | `packages/opencode/src/tool/shell.ts:347` |

## Evolution
- 2025-08-03 `21c52fd5cb`, 2025-10-12 `b4171aa8e8`: commands waiting on stdin hung forever → `stdin: "ignore"`.
- 2025-08-31 `029612d8d5` (#2339): abort killed only the shell → spawn detached, kill `-pid`.
- 2025-10-16 `fc18fc8a08` (#3225) "bash hangs & orphans": SIGTERM → 200 ms → SIGKILL.
- 2026-03-01 `c4c0b23bff` (#15516): MCP child processes orphaned on exit → kill descendants, expose `OPENCODE_PID`.
- 2026-03-24 `41c77ccb33`, 2026-04-01 `e4ff1ea778`: bash moved to Effect `ChildProcess` with `forceKillAfter`.

## Quirks / drift
- Two grace periods coexist: 3 s for tool shells, 200 ms for the core `killTree` helper.
- MCP descendant cleanup sends only SIGTERM, no KILL escalation (`packages/opencode/src/mcp/index.ts:546`).
- `pgrep` walk is POSIX-only; Windows MCP children rely on `client.close()` (inferred).

pi contrast: pi kills the group with SIGKILL immediately and sweeps tracked pids on shutdown ([[pi--process-tree-kill|pi]]).
