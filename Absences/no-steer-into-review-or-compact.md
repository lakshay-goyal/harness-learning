---
type: absence
harnesses: [codex]
---
# no-steer-into-review-or-compact

User input cannot steer an active Review or Compact task; clients must queue it for the next turn.

**What's missing**
- `NotSubmittedReason::ActiveTurnNotSteerable` for Review/Compact tasks (`codex-rs/core/src/session/turn_input.rs:824-835`); ReviewTask and CompactTask are not steerable (`codex-rs/core/src/session/turn_input.rs:830-833`).

**Evidence of decision**
- TUI queues follow-ups during manual `/compact`: `e838645fa2` 2026-03-23 "tui: queue follow-ups during manual /compact (#15259)".

**Implication**
- Steering is a property of the regular turn loop only; one-shot internal tasks are atomic from the user's view ([[single-active-task-slot]]).

Related: [[steering-queue]] · [[follow-up-queue]] · [[single-active-task-slot]] · [[review-subagent]] · [[auto-compaction]] · [[Absences]]
