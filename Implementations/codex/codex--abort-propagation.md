---
type: implementation
harness: codex
concept: abort-propagation
commit: 622e9e3696
files: [codex-rs/core/src/session/handlers.rs:492, codex-rs/core/src/session/mod.rs:5006, codex-rs/core/src/tasks/mod.rs:71, codex-rs/core/src/tasks/mod.rs:925, e95abcdf49:codex-rs/core/src/tools/parallel.rs:262, codex-rs/core/src/context/turn_aborted.rs:10, codex-rs/async-utils/src/lib.rs, codex-rs/protocol/src/protocol.rs:4306]
---
[[abort-propagation]] in [[codex]].

## Mechanism
1. **Entry**: `Op::Interrupt` handled inline on the submission loop (`codex-rs/core/src/session/handlers.rs:492-495`) → `Session::interrupt_task` → `abort_all_tasks(TurnAbortReason::Interrupted)`; if no turn is active it cancels MCP startup instead (`codex-rs/core/src/session/mod.rs:5006-5013`). TUI: Ctrl+C / Esc; app-server `turn/interrupt`.
2. **Conditional interrupt**: `Op::InterruptIfNoPendingInput{turn_id, reply}` interrupts only if that turn is still active AND nothing is queued (`codex-rs/core/src/session/handlers.rs:496-500`; `04fc75adbe` 2026-09-22 #47340).
3. **Token tree**: each `RunningTask` owns a root `CancellationToken`; the task gets `child_token()` (`codex-rs/core/src/tasks/mod.rs:315`, `:347`); `run_turn` passes `cancellation_token.child_token()` to each sampling request (`codex-rs/core/src/session/turn.rs:536`), which passes another child to each tool call (`codex-rs/core/src/stream_events_utils.rs:349`). Every await is wrapped `.or_cancel(&token)` — `OrCancelExt` = `tokio::select!` between the future and `token.cancelled()` (`codex-rs/async-utils/src/lib.rs`). Retry backoff sleeps are cancellable too (`codex-rs/core/src/session/turn.rs:1705-1729`).
4. **`handle_task_abort`** (`codex-rs/core/src/tasks/mod.rs:925-1029`): cancel token → optionally interrupt code-mode cells (`Feature::CodeModeInterrupt`, `:942-952`) → wait up to `GRACEFULL_INTERRUPTION_TIMEOUT_MS` = 100 ms for the task's `done` Notify (`:71`, `:961`) → hard `handle.abort()` → task-specific `abort()` hook → record model-visible interrupted marker and FLUSH rollout before emitting the event ("some clients synchronously re-read the rollout on receipt of the abort event", `:972-990`) → await extension `on_turn_abort` (`:1010`) → run Interrupt hooks (`:993-995`) → `EventMsg::TurnAborted{reason, started_at, completed_at, duration_ms}` → flush again ("buffering thread writers may not flush it without another explicit barrier", `:1023-1028`).
5. **Teardown order**: tasks observe cancellation BEFORE pending approvals are dropped, "or an in-flight approval wait can surface as a model-visible rejection before TurnAborted" (`codex-rs/core/src/tasks/mod.rs:569-572`); `input_queue.clear_pending` afterwards (`:620-622`) clears pending waiters AND pending input items — un-drained steers are dropped (`codex-rs/core/src/session/input_queue.rs:316-320`) → [[approval-wait-surfaces-as-rejection-on-interrupt]].
6. **Tools**: each call `tokio::spawn`ed under `AbortOnDropHandle` (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:196`); readiness wait, lock acquisition and PreToolUse hook waits are cancellable (`:199-218`; `codex-rs/core/src/tools/registry.rs:626-678`). On cancellation the runtime aborts the task unless it `finishes_on_cancellation()` or already reached a terminal outcome, then synthesizes an output: `"Wall time: {secs:.1} seconds\naborted by user"` for `exec_command`, `"aborted by user after {secs:.1}s"` otherwise (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:262-301`, `:363-380`) → call/output pairing preserved ([[transcript-replay-repair]]). Dynamic (client-supplied) handlers publish their own terminal item (`:135-305`; `18e28fe1b9`).
7. **Background shells survive**: interrupt leaves unified-exec processes running; explicit `/stop` → `Op::CleanBackgroundTerminals` kills them (`codex-rs/core/src/session/handlers.rs:501-504`, `codex-rs/core/src/tasks/mod.rs:907-912`) → [[interrupt-kills-background-processes]].
8. **Model-visible marker**: `INTERRUPTED_GUIDANCE` = "The user interrupted the previous turn on purpose. Any running unified exec processes may still be running in the background. If any tools/commands were aborted, they may have partially executed." (`codex-rs/core/src/context/turn_aborted.rs:10`), wrapped `<turn_aborted>…</turn_aborted>` (`:34`), content kind `generic.turn_aborted` (`:22`). Marker role configurable: contextual-user, developer, or disabled (`codex-rs/core/src/tasks/mod.rs:102-118`; `agents.interrupt_message`, `28742866c7`) → [[interrupted-turn-invisible-to-model]].
9. **Abort reasons**: `Interrupted`, `Replaced` (new task over old: `spawn_task` calls `abort_all_tasks(Replaced)` first, `codex-rs/core/src/tasks/mod.rs:272-281`), `ReviewEnded`, `BudgetLimited` (`codex-rs/protocol/src/protocol.rs:4306-4311`); `Interrupted|BudgetLimited` mark the session interrupted (`codex-rs/core/src/tasks/mod.rs:594`).
10. **After abort**: `maybe_start_turn_for_pending_work` may immediately start a new turn for queued agent mail (`codex-rs/core/src/tasks/mod.rs:574-576`); the durable user queue does NOT dispatch after an interrupt (`on_thread_idle` returns early on `ThreadIdleCause::Interrupted`, `codex-rs/ext/queue/src/service.rs:549-553`) → [[follow-up-queue]].
11. **Suspend ≠ abort**: `Op::SuspendTurnAndShutdown` flushes BEFORE cancelling, cancels without a terminal event (another worker may recover the same turn id), same 100 ms grace, drops in-process pending input (`codex-rs/core/src/session/turn_suspension.rs:1-119`) → [[durable-execution]].
12. **Errors**: `CodexErr::TurnAborted` returned by a task routes through the aborted-turn lifecycle (`codex-rs/core/src/tasks/mod.rs:175-216`); `retry_delay` → `None` for TurnAborted/Interrupted (`codex-rs/protocol/src/error.rs:397-424`).

## Constants
| name | value | path:line |
|---|---|---|
| graceful task-abort wait before hard abort | `GRACEFULL_INTERRUPTION_TIMEOUT_MS` = 100 ms | `codex-rs/core/src/tasks/mod.rs:71` |
| aborted tool output (exec_command) | `"Wall time: {secs:.1} seconds\naborted by user"` | `e95abcdf49:codex-rs/core/src/tools/parallel.rs:363-380` |
| Guardian reviewer interrupt drain | `GUARDIAN_INTERRUPT_DRAIN_TIMEOUT` = 5 s | `codex-rs/ext/guardian-reviewer/src/execution.rs:20` |
| user shell command default timeout | `USER_SHELL_TIMEOUT_MS` = 3 600 000 | `codex-rs/core/src/tasks/user_shell.rs:47` |

## Evolution
- 2025-08-17 `b581498882` `EventMsg::TurnAborted` introduced.
- 2025-10-17 `c03e31ecf5` "Support graceful agent interruption (#5287)" — 100 ms grace.
- 2025-10-23 `f59978ed3d` cancellation handled while processing a turn; items recorded even on abort (#5543) → [[turn-items-lost-on-abort]].
- 2026-01-20 `b236f1c95d` model-visible `<turn_aborted>` marker (#9043); text softened `09251387e0` 2026-01-26, `2b8d29ac0d` 2026-03-31.
- 2026-02-03 `e3d39013d3` headless `codex exec` exits on Interrupt with websockets.
- 2026-03-09 `ad57505ef5` interrupted-task approval cleanup ordering (#14102).
- 2026-03-12 `d9a403a8c0` hard-stop js_repl execs on interrupt (#13329).
- 2026-03-15 `ba463a9dc7` background terminals preserved on interrupt; cleanup renamed `/stop` (#14602).
- 2026-03-17 `6ea041032b` interrupt allowed during WebSocket warm-up (#14838).
- 2026-04-24 `11806faf71` `turn/interrupt` RPC no longer hangs on a finished turn (#18392); `28742866c7` `agents.interrupt_message`; `120aa07d81` MultiAgentV2 markers non-user-authored.
- 2026-07-15 `70a0b1eef8` output-free interrupted prompt kept in transcript.
- 2026-07-22 `d7e8f4c3dc` user input preserved when MCP startup is interrupted.
- 2026-08-07 `509565820f` interrupt active code-mode cells with their turn (#37483).
- 2026-08-25 `cbfd999db7` Interrupt hooks (#40511); 2026-08-28 `c2abf869d5` executor hooks for interrupted turns (#41432).
- 2026-09-22 `04fc75adbe` `Op::InterruptIfNoPendingInput` (#47340).
- 2026-09-30 `9ef9cb1d9f` await extension `on_turn_abort` before `TurnAborted` (#49475).
- 2026-10-07 `18e28fe1b9` dynamic tools settle lifecycle on cancellation (#51556).

## Quirks
- Grace is short (100 ms): interrupt feels instant, but tasks get little time to record their own outputs — synthetic aborted outputs fill the gap.
- `Op::Interrupt` on an idle session is a no-op in core (it cancels MCP startup) — RPC layers must answer anyway ([[interrupt-rpc-hangs-on-finished-turn]]).
- Guardian denial loops end child turns with `TooManyDenials`, not an abort ([[delegated-authorization-provenance]]).

## Versus pi
- [[pi--abort-propagation]]: one `AbortController` per run + signal-state classification vs codex token tree + `.or_cancel` per await + grace-then-hard-abort.
- pi kills the bash process tree ([[process-tree-kill]]); codex deliberately leaves background unified-exec processes running and tells the model.
- pi keeps the aborted partial and skips it on replay; codex writes synthetic tool outputs immediately plus a `<turn_aborted>` marker.
