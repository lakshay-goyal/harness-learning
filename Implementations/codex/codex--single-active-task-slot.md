---
type: implementation
harness: codex
concept: single-active-task-slot
commit: 622e9e3696
files: [codex-rs/core/src/tasks/mod.rs:175, codex-rs/core/src/tasks/mod.rs:272, codex-rs/core/src/tasks/mod.rs:287, codex-rs/core/src/tasks/regular.rs:37, codex-rs/core/src/tasks/compact.rs:63, codex-rs/core/src/tasks/review.rs:99, codex-rs/core/src/tasks/user_shell.rs:48, codex-rs/core/src/session/turn_input.rs:824]
---
[[single-active-task-slot]] in [[codex]].

## Mechanism
- **Trait** `SessionTask`: `kind()`, `span_name()`, `run(session, ctx, input, cancellation_token) -> Result<Option<String>>`, optional `abort()` cleanup; returning `CodexErr::TurnAborted` routes through the aborted-turn lifecycle (`codex-rs/core/src/tasks/mod.rs:175-216`).
- **Replace semantics**: `spawn_task` = `abort_all_tasks(TurnAbortReason::Replaced)` + `start_task` (`codex-rs/core/src/tasks/mod.rs:272-281`).
- **`start_task`**: drains the mailbox into the new turn's pending input, records the active reservation, spawns the future under `AbortOnDropHandle`, and on normal completion flushes the rollout then `on_task_finished` (`codex-rs/core/src/tasks/mod.rs:287-425`) → [[run-settlement]].
- **RegularTask**: TurnStarted inline (first turn doesn't wait on prewarm), cancellable turn-start extension contributors, consumes startup WebSocket prewarm, loops `run_turn` while pending input remains, stops on recorded terminal error (`codex-rs/core/src/tasks/regular.rs:37-124`) → [[turn-loop]].
- **CompactTask** (manual `Op::Compact`, `codex-rs/core/src/session/handlers.rs:689`): remote compaction v2 if provider supports it, else local summarization with `config.compact_prompt` or `SUMMARIZATION_PROMPT` (`codex-rs/core/src/tasks/compact.rs:63-95`) → [[auto-compaction]]. Not steerable (`codex-rs/core/src/session/turn_input.rs:830-833`).
- **ReviewTask**: one-shot child Codex thread with `REVIEW_PROMPT` as base instructions, web search, Collab and MultiAgentV2 disabled, approval policy `Never`, optional `review_model` (`codex-rs/core/src/tasks/review.rs:99-135`); JSON result parse with first-`{…}` fallback (`:190-210`); not steerable; ends with `TurnAbortReason::ReviewEnded` → [[review-subagent]].
- **UserShellCommandTask**: if a turn is active the command runs as an auxiliary of that turn (shares its cancellation token, no second TurnStarted/TurnComplete), otherwise a standalone task (`codex-rs/core/src/session/handlers.rs:99-131`, `codex-rs/core/src/tasks/user_shell.rs:48-57`); its `kind()` reports `TaskKind::Regular` (`codex-rs/core/src/tasks/user_shell.rs:75-77`) → [[user-shell-escape]].
- **Steer gate**: Review and Compact are `ActiveTurnNotSteerable` (`codex-rs/core/src/session/turn_input.rs:824-835`) → [[no-steer-into-review-or-compact]].
- **Suspend** only for a Regular root turn with no live descendants (`codex-rs/core/src/session/turn_suspension.rs:1-119`) → [[durable-execution]].

## Constants
| name | value | path:line |
|---|---|---|
| user shell command default timeout | `USER_SHELL_TIMEOUT_MS` = 3 600 000 (1 h) | `codex-rs/core/src/tasks/user_shell.rs:47` |
| graceful abort before hard abort | 100 ms | `codex-rs/core/src/tasks/mod.rs:71` |

## Evolution
- 2025-10-28 `5ba2a17576` "chore: decompose submission loop (#5854)".
- 2025-10-29 `13e1d0362d` review delegated to a child Codex instance (ReviewTask).
- 2026-03-15 `ba463a9dc7` `/stop` → `Op::CleanBackgroundTerminals` separate from interrupt.
- 2026-03-23 `e838645fa2` TUI queues follow-ups during manual `/compact`.
- 2026-04-16 `a1736fcd20` "[codex] Split codex turn logic (#18206)".

## Quirks
- User shell commands masquerade as `TaskKind::Regular` and piggyback on an active turn's token — an interrupt kills both.

## Versus pi
- pi rejects `prompt()` during a run and keeps compaction / branch summary as separately aborted side phases (`abortCompaction`, `abortBranchSummary`, [[pi--abort-propagation]]); codex funnels every kind of work through one slot with one lifecycle and replaces rather than rejects.
