---
type: failure
concepts: [task-owned-subagent]
harnesses: [opencode]
---
**Symptom** — With a `task_status(task_id, wait, timeout_ms)` tool for background subagents, the model slept, polled for progress and re-checked status, burning turns instead of doing other work.

**Root cause** — A status tool invites busy-waiting; the model has no other way to learn completion.

**Fix · [[opencode]]** — `dabf2dc013` 2026-05-25 "remove the need for polling from experimental background agents" (#29179) deleted `task_status` (179 lines; `dabf2dc013^:packages/opencode/src/tool/task_status.txt`); completion is pushed as a synthetic user message "Background task completed: …" that starts a new parent turn (`packages/opencode/src/tool/task.ts:227-264`). `cc9b73b0bd` 2026-06-04 (#30790) added to the tool result "Do not poll for progress, ask the task for status, or duplicate this task's work"; models still slept between checks, so `b9131aa69c` 2026-06-06 (#31162) "lets kill this sleep behavior" escalated it to "DO NOT sleep, poll for progress, …" (`packages/opencode/src/tool/task.ts:31-35`).

**Lesson** — Async work should notify the model by injecting a message, not offer a poll tool.

Related: [[task-owned-subagent]] · [[follow-up-queue]] · [[delegated-work-duplicated]] · [[opencode--task-owned-subagent|opencode]]
