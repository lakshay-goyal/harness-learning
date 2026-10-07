---
type: implementation
harness: codex
concept: run-settlement
commit: 622e9e3696
files: [codex-rs/core/src/tasks/regular.rs:37, codex-rs/core/src/tasks/regular.rs:110, codex-rs/core/src/tasks/mod.rs:357, codex-rs/core/src/tasks/mod.rs:672, codex-rs/core/src/tasks/mod.rs:888, codex-rs/core/src/tasks/mod.rs:1023, codex-rs/ext/goal/src/extension.rs:180, codex-rs/ext/queue/src/service.rs:549]
---
[[run-settlement]] in [[codex]].

## Mechanism
1. **Task wrapper as driver**: `RegularTask` emits TurnStarted, runs turn-start extension contributors, consumes the prewarm, then loops `run_turn` while pending input remains, stopping if `ctx.terminal_error` was recorded (`codex-rs/core/src/tasks/regular.rs:37-124`, `:110-119`). One task = possibly several `run_turn`s.
2. **Inside `run_turn`** the settle decision is `!needs_follow_up` → Stop hooks (may re-open) → legacy after-agent hook → optional post-turn compaction (skipped with TokenBudget) → `break` (`codex-rs/core/src/session/turn.rs:622-725`, `:684-686`).
3. **Leftover input**: pending input at task end is recorded into history (through hooks) for a later explicit request instead of being dropped (`codex-rs/core/src/tasks/mod.rs:672-697`).
4. **Persistence barrier**: on normal completion the rollout is flushed BEFORE `on_task_finished` (`codex-rs/core/src/tasks/mod.rs:357-374`); after emitting `TurnAborted`/`TurnComplete` flush again ("buffering thread writers may not flush it without another explicit barrier", `:1023-1028`) → [[listeners-see-stale-agent-state]].
5. **Post-settle re-check**: after clearing the active turn, `maybe_start_turn_for_pending_work` starts a new turn for queued agent mail (`codex-rs/core/src/tasks/mod.rs:888-890`, `:456-546`; re-checks reservation after async work).
6. **Idle extensions**: `on_thread_idle(ThreadIdleInput{cause, …})` lets extensions continue work: durable queue pops head → `start_turn_if_idle(turn_trigger:"queue")`, skipped when `cause == ThreadIdleCause::Interrupted` (`codex-rs/ext/queue/src/service.rs:405-470`, `:549-553`); goal extension `continue_if_idle` (`codex-rs/ext/goal/src/extension.rs:180-192`) → [[follow-up-queue]], [[persistent-goal-continuation]]. `start_turn_if_idle` is rejected (not injected) if a turn is active (`f1b1b64005`).
7. **Single slot**: starting any task aborts the current one (`TurnAbortReason::Replaced`) → [[single-active-task-slot]].
8. **Reply semantics**: `Op::TurnInput{reply}` answers the routing decision, not the turn outcome; "Once queued, dropping the waiter does not retract the call. If the session loop exits before replying, the caller gets InternalAgentDied" (`codex-rs/core/src/session/mod.rs:1053-1060`) — settlement is observed via events ([[agent-event-stream]]).
9. **Pre-step ordering**: pre-turn compaction runs before input is recorded; on failure the input is recorded and prompt hooks run before the error is reported (`codex-rs/core/src/session/turn.rs:184-224`; `codex-rs/core/src/compact.rs:250` "Pre-turn failures are reported after preserving the incoming prompt") → [[compaction-drops-pending-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| idle-dispatch triggers | `"queue"`, `"goal"` (`turn_trigger`) | `codex-rs/ext/queue/src/service.rs:405-470`; `codex-rs/ext/goal/src/runtime.rs:425-523` |

## Evolution
- 2026-01-28 `26590d7927` "Ensure auto-compaction starts after turn started (#10129)" → [[compaction-cancellation-races]].
- 2026-04-02 `e47ed5e57f` "fix: races in end of turn (#16566)".
- 2026-06-01 `f1b1b64005` "Add goal extension idle continuation" — idle-only turn start, fail instead of inject.
- 2026-08-06 `bc8b25ea02` durable queue idle dispatch.
- 2026-08-25 `d7510aa4b4` "Preserve pending input after terminal turn errors (#40613)" → [[pending-input-restarts-failed-turn]].
- 2026-09-10 `ee93abb690` preserve incoming prompts when pre-turn compaction fails (#44487).
- 2026-09-30 `9ef9cb1d9f` await extension `on_turn_abort` before `TurnAborted`.

## Versus pi
- [[pi--run-settlement]]: pi has an explicit `agent_settled` event and a before-settle extension hook allowing one continuation; codex has no single settled event — idle = no active task, observed through `TurnComplete`/`TurnAborted` plus extension `on_thread_idle`, and continuation drivers are extensions that start fresh turns.
- Both make persistence part of settlement (pi awaited listeners, codex explicit flush barriers).
