---
type: concept
stage: loop
tier: variant
aliases: [SessionTask, AnySessionTask, "TaskKind::Regular", "TaskKind::Review", "TaskKind::Compact", RunningTask, ActiveTurn, spawn_task, start_task, RegularTask, "TurnAbortReason::Replaced"]
harnesses: [codex]
---
A session runs at most one "task" at a time (regular turn, manual compaction, review, user shell); starting any task first aborts the current one, and every task kind shares one start / finish / abort lifecycle and the same terminal events.

## Why
- Side phases (compaction, review, user shell) are second "runs"; without one slot they race the main loop and each other ([[compaction-cancellation-races]]).
- One lifecycle means one place for abort grace, persistence barriers and terminal events — clients see the same `TurnStarted`/`TurnComplete`/`TurnAborted` for every kind ([[listeners-see-stale-agent-state]]).
- Input routing becomes a single question "is a task active and steerable?" ([[steering-queue]]).

## Design space
- **Concurrency**: one slot, new task replaces old (✔ codex `spawn_task` = `abort_all_tasks(Replaced)` + `start_task`) · reject new work while busy (pi `prompt()` rejection, [[turn-loop]]) · queue new work.
- **Kinds**: regular / compact / review / user-shell share a trait (`run`, optional `abort` cleanup) (✔ codex) · ad-hoc code paths per side phase (pi `abortCompaction`, `abortBranchSummary`).
- **Steerability per kind**: regular steerable; review/compact reject steers (✔ codex `ActiveTurnNotSteerable`).
- **Auxiliary work**: user shell command joins the active turn as an auxiliary sharing its cancellation token (no second TurnStarted) or runs standalone when idle (✔ codex, [[user-shell-escape]]).
- **Multiple turns per task**: task re-runs the turn while input is pending (✔ codex `RegularTask`).

## Implementations
- [[codex--single-active-task-slot|codex]] — `SessionTask` trait + `RunningTask` slot; `spawn_task` aborts with `Replaced`; Regular / Compact / Review / UserShellCommand tasks.

## Failures
- [[prewarm-blocks-turn-start]]
- [[pending-input-restarts-failed-turn]]
- [[compaction-cancellation-races]]

## Related
[[turn-loop]] · [[run-settlement]] · [[abort-propagation]] · [[steering-queue]] · [[auto-compaction]] · [[review-subagent]] · [[user-shell-escape]] · [[agent-event-stream]]
