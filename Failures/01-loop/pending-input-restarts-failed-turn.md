---
type: failure
concepts: [follow-up-queue, steering-queue, turn-loop]
harnesses: [codex]
---
**Symptom** — "Pending user input or agent mail could cause a regular task to restart immediately after a terminal compaction error, retrying the failed turn" (`d7510aa4b4` body).

**Root cause** — The task wrapper's continue condition ("input is pending → run another turn") ignored that the previous turn had failed terminally.

**Fix · [[codex]]**
- `d7510aa4b4` 2026-08-25 "Preserve pending input after terminal turn errors (#40613)": `RegularTask` stops its `run_turn` loop when `ctx.terminal_error` is set (`codex-rs/core/src/tasks/regular.rs:110-119`); pending input is persisted into history at task completion for a later explicit request (`codex-rs/core/src/tasks/mod.rs:672-697`).

**Lesson** — "Input is pending" must not override "this turn failed terminally"; otherwise queued input becomes an unbounded retry loop.

Related: [[turn-loop]] · [[run-settlement]] · [[follow-up-queue]] · [[steering-queue]] · [[single-active-task-slot]] · [[goal-continuation-runaway]] · [[codex--run-settlement|codex]]
