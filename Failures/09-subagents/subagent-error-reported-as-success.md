---
type: failure
concepts: [task-owned-subagent, tool-error-as-result]
harnesses: [opencode]
---
**Symptom** — When a subagent's run errored (assistant error, or its last tool call failed), the `task` tool still returned the child's last text as a normal result; the parent model believed the work was done.

**Root cause** — The task result was derived from child text only, not from the child's terminal state.

**Fix · [[opencode]]** — `c313504c82` 2026-08-20 "surface resumable subagent errors" (#43657) and `35fe5b7212` 2026-08-21 "surface subagent tool errors" (#43821): the task fails with `Subagent failed (task_id: …): <error>` when the child message has an error or its last tool part errored, keeping the resumable `task_id` (`packages/opencode/src/tool/task.ts:213-224`).

**Lesson** — Propagate the child's terminal state, not just its text, and give the parent a handle to resume.

Related: [[task-owned-subagent]] · [[tool-error-as-result]] · [[subagent-output-truncated]] · [[opencode--task-owned-subagent|opencode]]
