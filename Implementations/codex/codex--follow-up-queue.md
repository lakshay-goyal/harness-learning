---
type: implementation
harness: codex
concept: follow-up-queue
commit: 622e9e3696
files: [codex-rs/ext/queue/src/lib.rs:1, codex-rs/ext/queue/src/service.rs:265, codex-rs/ext/queue/src/service.rs:405, codex-rs/ext/queue/src/service.rs:549, codex-rs/core/src/session/handlers.rs:450, codex-rs/core/src/tasks/mod.rs:456, codex-rs/core/src/session/input_queue.rs:322, codex-rs/core/src/tasks/mod.rs:672]
---
[[follow-up-queue]] in [[codex]].

## Mechanism
1. **No core user follow-up queue**: the TUI queues with Tab and submits on idle (`cbca43d57a` 2026-01-12 "Send message by default mid turn. queue messages by tab"). Inside core the "follow-up" analog is the agent MAILBOX: `submission_loop` also selects on a mailbox watch channel and calls `maybe_start_turn_for_pending_work` when the thread is durably asleep (`codex-rs/core/src/session/handlers.rs:450-479`).
2. **Idle start for mail**: `maybe_start_turn_for_pending_work_with_sub_id` starts a fresh `RegularTask` when mailbox items exist; queue-only mail (`trigger_turn == false`) reuses previous turn options; re-checks the reservation after async work so an interrupt can't be undone (`codex-rs/core/src/tasks/mod.rs:456-546`). Called after an interrupt (`:574-576`) and after clearing the active turn (`:888-890`).
3. **Final-answer boundary**: after a final-answer assistant message, queue-only mail is deferred to the next turn (`MailboxDeliveryPhase::NextTurn`) instead of reopening sampling (`codex-rs/core/src/session/input_queue.rs:322-343`, `codex-rs/core/src/stream_events_utils.rs:529-544`); re-accepted for the current turn when a tool call / Stop-hook continuation / injection reopens it (`codex-rs/core/src/session/input_queue.rs:322-366`, `codex-rs/core/src/stream_events_utils.rs:333-338`, `codex-rs/core/src/session/turn.rs:545-552`).
4. **Mid-stream mailbox preemption**: after reasoning or commentary/partial-answer items, pending mail cuts the sampling request short with `needs_follow_up: true` (`codex-rs/core/src/session/turn.rs:2747-2770`, `:2786-2811`); opt-out flag `DeferMailboxPreemption` (`codex-rs/features/src/lib.rs:1446-1451`, under development, default off).
5. **Durable user-message queue extension** `codex-rs/ext/queue` — "Durable, storage-neutral user-message queue and idle dispatch" (`codex-rs/ext/queue/src/lib.rs:1`): enqueue/list/update/delete/reorder API (`codex-rs/ext/queue/src/service.rs:265-368`); on each `on_thread_idle` pops the head and calls `start_turn_if_idle(... turn_trigger: "queue")`, deleting the item only once the turn Started (`:405-470`). Non-user or undecodable items discarded with a warning (`:425-437`). Interrupt does NOT drain: `on_thread_idle` returns early when `cause == ThreadIdleCause::Interrupted` (`:549-553`). Storage: `queue_1.sqlite` ([[sqlite-session-index]]). App-server thread queue APIs (`9341b38310`).
6. **Leftover input**: pending input at task end is recorded into history (through hooks) rather than dropped (`codex-rs/core/src/tasks/mod.rs:672-697`); after a terminal error the `RegularTask` stops instead of re-running the failed turn for pending input (`codex-rs/core/src/tasks/regular.rs:110-119`) → [[pending-input-restarts-failed-turn]].

## Constants
| name | value | path:line |
|---|---|---|
| max pending queued submissions per thread | 100 | `codex-rs/state/src/lib.rs:113` |
| `turn_trigger` for queue dispatch | `"queue"` | `codex-rs/ext/queue/src/service.rs:405-470` |
| `DeferMailboxPreemption` | under development, default off | `codex-rs/features/src/lib.rs:1446-1451` |

## Evolution
- 2026-01-12 `cbca43d57a` TUI Tab queue.
- 2026-04-02 `e47ed5e57f` "fix: races in end of turn (#16566)".
- 2026-04-03 `e4f1b3a65e` mid-stream mailbox preemption (#16725); 2026-04-13 `05c5829923` "drain mailbox only at request boundaries"; 2026-04-18 `e3c2acb9cd` reverted (#18325); 2026-09-24 `766d2377a8` opt-in defer flag (#47913).
- 2026-07-15 `c28770a42f` "Respect final-answer boundaries for queued agent mail (#33367)" → [[queued-messages-stranded-at-run-end]].
- 2026-08-06 `bc8b25ea02` "Add durable user-message queue dispatch (#37204)"; 2026-08-12 `da2803c73c` "Simplify queued user message admission (#38092)"; 2026-08-13 `9341b38310` "Add experimental thread queue APIs to app server (#38456)".
- 2026-08-25 `d7510aa4b4` "Preserve pending input after terminal turn errors (#40613)".

## Quirks
- Two idle drivers share the same `on_thread_idle` seam: queue dispatch and goal continuation ([[persistent-goal-continuation]]); both use `start_turn_if_idle` and are rejected (not injected) when a turn is active.
- Mid-stream mail preemption was removed and restored within 5 days — preemption won over strict request-boundary delivery.

## Versus pi
- [[pi--follow-up-queue]]: pi's follow-up queue is in-memory and drained at the loop's natural stop; codex's is a persisted extension that starts a new turn when the thread goes idle (survives restarts) and never fires after an interrupt.
- pi re-checks queues after end-of-run handlers; codex re-checks mailbox after clearing the active turn.
