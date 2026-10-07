---
type: failure
concepts: [subagent-result-mailbox, steering-queue]
harnesses: [codex]
---
**Symptom** — A long `wait_agent` held the turn; a user's steer message stayed pending until the wait returned, so redirects looked ignored.

**Root cause** — The blocking tool waited only for mailbox activity or timeout; user input was a separate queue drained at the next loop boundary, which the wait itself was blocking.

**Fix · [[codex]]**
- `ee40dddbf6` 2026-06-15 "core: let steer interrupt wait_agent" — steer wakes waiters, result "Wait interrupted by new input." (`codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs` `WaitOutcome::Steered`); description: "The wait also ends early when new user input is steered into the active turn." (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:306`). Same contract for `clock.sleep` ("ends early when new input arrives for the active turn", [[wall-clock-tools]]).

**Lesson** — Blocking tools must be interruptible by user input, not just by abort.

Related: [[subagent-result-mailbox]] · [[steering-queue]] · [[retry-backoff-hygiene]] · [[wall-clock-tools]] · [[codex--subagent-result-mailbox|codex]]
