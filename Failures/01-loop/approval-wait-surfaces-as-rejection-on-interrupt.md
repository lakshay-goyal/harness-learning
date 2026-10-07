---
type: failure
concepts: [abort-propagation, tool-call-gate]
harnesses: [codex]
---
**Symptom** — On interrupt, pending approvals were cleared before the running task observed cancellation, so "an in-flight approval wait [could] surface as a model-visible rejection before the TurnAborted" — the model saw "user declined" for an action the user never judged.

**Root cause** — Teardown order: the resource a consumer was waiting on (approval channel) was destroyed before the consumer's cancellation, and the waiter mapped "channel closed" to a denial.

**Fix · [[codex]]**
- `ad57505ef5` 2026-03-09 "Stabilize interrupted task approval cleanup (#14102)": cancel and drain the task first, then `input_queue.clear_pending` (`codex-rs/core/src/tasks/mod.rs:569-572`, `:620-622`; comment "or an in-flight approval wait can surface as a model-visible rejection before TurnAborted").

**Lesson** — On abort, cancel consumers before tearing down the resources they wait on, or teardown looks like a user decision.

Related: [[abort-propagation]] · [[tool-call-gate]] · [[approval-policy-modes]] · [[codex--abort-propagation|codex]]
