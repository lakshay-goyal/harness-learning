---
type: failure
concepts: [shell-execution, task-owned-subagent]
harnesses: [opencode]
---
**Symptom** — Background work (background bash in v2, background subagents with a `task_status` tool in legacy) left the model without a reliable way to observe completion; it slept, polled, or lost track of jobs.

**Root cause** — An async capability was exposed without its observe/cancel pair, or with a poll tool that invited busy-waiting.

**Fix · [[opencode]]**
- Legacy: `dabf2dc013` 2026-05-25 `task_status` removed ("remove the need for polling"); completion pushed into the parent transcript → [[model-polls-background-work]].
- v2: background bash removed: "The model has no registered observation or cancellation tool for background bash jobs, and process-local status is not a sufficient remote contract"; reintroduce "only with durable status observation, completion delivery, and explicit cancellation semantics" (`specs/v2/schema-changelog.md:697,702`; prototype service kept, explicitly non-durable, `packages/core/src/background-job.ts`).

**Lesson** — Never expose an async capability without push-based completion and an explicit cancel.

Related: [[shell-execution]] · [[task-owned-subagent]] · [[no-background-bash]] · [[opencode--shell-execution|opencode]]
