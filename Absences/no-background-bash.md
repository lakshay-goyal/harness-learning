---
type: absence
harnesses: [pi, opencode]
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

**codex** — *present; the opposite design*. Unified exec makes long-lived PTY processes first-class: `exec_command` yields a session id after a yield window and the model continues with `write_stdin` (`c09ed74a16` 2025-09-10 "Unified execution"; 262 commits mention unified exec since). Background terminals survive interrupt (`ba463a9dc7` 2026-03-15; the `<turn_aborted>` marker tells the model "Any running unified exec processes may still be running in the background", `codex-rs/core/src/context/turn_aborted.rs:10-38`). Legacy one-shot `shell_command` removed `8a40095ea3` 2026-08-20 "Standardize shell execution on unified exec" → [[removed-legacy-shell-tools]]. Constants: `MIN_YIELD_TIME_MS = 250`, `MIN_EMPTY_YIELD_TIME_MS = 5_000`, `MAX_YIELD_TIME_MS = 30_000`, `DEFAULT_MAX_BACKGROUND_TERMINAL_TIMEOUT_MS = 300_000` (`codex-rs/core/src/unified_exec/mod.rs:73-78`); 64-process store (M3 findings). TUI `/ps`, `/stop`; `Op::CleanBackgroundTerminals`.
**opencode** ([[opencode]], `ecc4916b5a`): also absent for the model, and removed on purpose in v2.
- Legacy shell schema is `command`, `timeout`, `workdir` (+ description); no background/detach flag (`packages/opencode/src/tool/shell/prompt.ts:15-20`). Every call blocks under the 2-min default timeout (`packages/opencode/src/tool/shell.ts:347`).
- v2 removed background bash: "The model has no registered observation or cancellation tool for background bash jobs, and process-local status is not a sufficient remote contract" (`specs/v2/schema-changelog.md:697`; `d29f5eba92` 2026-06-22).
- What does run in the background: **subagents** only, behind `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS` (`task` param `background`, `packages/opencode/src/tool/task.ts:58-61`; `22de34c4de` 2026-05-14), on a registry that is "intentionally not durable" (`packages/core/src/background-job.ts:113-118`). Completion is pushed as a message, never polled → [[no-background-task-polling]].
- A PTY service exists for user terminals in the app (`packages/core/src/pty.ts`), not as a model tool.
- Same lesson from both sides: pi says use tmux; opencode says no async capability without its observe/cancel pair.

Related: [[shell-execution]] · [[process-tree-kill]] · [[abort-propagation]] · [[no-bash-default-timeout]] · [[no-turn-cap]] · [[Absences]] · [[codex--shell-execution|codex]] · [[no-kill-timeout-in-unified-exec]] · [[removed-legacy-shell-tools]]
Related: [[shell-execution]] · [[process-tree-kill]] · [[abort-propagation]] · [[no-bash-default-timeout]] · [[no-turn-cap]] · [[opencode]] · [[Absences]]
