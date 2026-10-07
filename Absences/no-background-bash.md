---
type: absence
harnesses: [pi]
---
# no-background-bash

**What's missing**
- The bash tool always runs to completion. There is no `run_in_background` / job id / output-polling tool, and no `nohup` handling.
- Schema is `command` plus optional `timeout` only, with no `cwd` param (`packages/coding-agent/src/core/tools/bash.ts:40-43`).
- No interactive input: stdin is `ignore`, or a pipe used only to feed the command text itself when `commandFromStdin` (`packages/coding-agent/src/core/tools/bash.ts:116`, inside `createLocalShellOperations` `:97-166`).

**Evidence of decision**
- `271810c80` (2025-11-12), README "Background Bash": "**pi does not and will not implement background bash execution.** Instead, tell the agent to use `tmux` or something like tterminal-cp. Long-running commands belong in proper terminal sessions, not as detached processes that complicate cleanup and monitoring."
- `25cc5c7bf^:packages/coding-agent/README.md:547`: "**No background bash.** Use tmux. Full observability, direct interaction." The section was deleted in `25cc5c7bf`, and the rationale is not restated at HEAD.

**Mechanism that enforces it** → [[shell-execution]], [[process-tree-kill]]
- Spawn uses `detached: true` on non-Windows to get a process group (`packages/coding-agent/src/core/tools/bash.ts:114`).
- Abort/timeout kills the whole group: `process.kill(-pid, SIGKILL)`, or `taskkill /F /T` on Windows (`packages/coding-agent/src/utils/shell.ts:187-218`). Origin `6e9fa8dde` (2025-11-11): `exec()` had killed only the parent shell, so `sleep 4 && echo hello` survived an abort.
- Detached pids are tracked in a module Set (`packages/coding-agent/src/core/tools/bash.ts:123,160`; `shell.ts:167-182`). `killTrackedDetachedChildren()` kills them on SIGHUP/SIGTERM/exit in interactive/print/rpc modes (`9b7948c4c`; `modes/print-mode.ts:58`, `modes/rpc/rpc-mode.ts:374`).
- Windows: descendants inheriting stdout kept `close` from firing. Fixed by resolving on `exit` plus pipe idle (`25b185f39`, #2389; `EXIT_STDIO_GRACE_MS = 100`, `utils/child-process.ts:16`).
- Open question (unverified): whether `cmd &` grandchildren are reaped at tool completion or only at pi shutdown.

**Opt-in replacement**
- tmux / tterminal-cp, driven by the model through ordinary bash.
- `examples/extensions/interactive-shell.ts` runs vim/htop etc. with the TUI suspended, via the `user_bash` hook. This is for user `!` commands, not model background jobs.
- `docs/tmux.md` covers only running pi inside tmux.
- No background-bash example exists (`08-absences` example map).
- Durable is the contrast: background *tasks* exist as a runtime primitive (background tasks are abort/idle boundaries, `packages/durable/docs/spec.md:1958-1965`), but durable `bash` also has no background mode (`packages/durable/src/tools/bash.ts`).

**History**
- Never reversed. Related default flip: bash default timeout 30s → none (`29900ce64`, 2025-11-12, same day) → [[no-bash-default-timeout]].

**Implication**
- Long-running servers, watchers and dev servers must live outside the agent (tmux) or block the turn. Cleanup and observability stay simple: anything pi spawns dies with pi.
- Pairs with no default timeout. A blocking `npm run dev` hangs the turn until the user presses Esc ([[abort-propagation]]).

Related: [[shell-execution]] · [[process-tree-kill]] · [[abort-propagation]] · [[no-bash-default-timeout]] · [[no-turn-cap]] · [[Absences]]
