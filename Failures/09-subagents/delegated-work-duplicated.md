---
type: failure
concepts: [task-owned-subagent]
harnesses: [opencode]
---
**Symptom** — After delegating work to a subagent, the parent model redid the same searches or edits itself (and, with background subagents, touched the same files concurrently).

**Root cause** — Nothing in the tool contract told the model that delegated work is owned by the child.

**Fix · [[opencode]]** — prompt only: `70bb710715` 2026-06-04 task description "Once you have delegated work to an agent, do not duplicate that work yourself" (`packages/opencode/src/tool/task.txt`); background start/update results say "DO NOT … duplicate this task's work — avoid working with the same files or topics it is using" (`packages/opencode/src/tool/task.ts:31-41`).

**Lesson** — Delegation needs an ownership rule stated in the tool contract; for concurrent children, name the files/topics the parent must leave alone.

Related: [[task-owned-subagent]] · [[model-polls-background-work]] · [[tool-description-design]] · [[opencode--task-owned-subagent|opencode]]
