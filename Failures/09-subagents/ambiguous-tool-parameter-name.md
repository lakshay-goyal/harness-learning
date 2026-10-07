---
type: failure
concepts: [task-owned-subagent, tool-description-design]
harnesses: [opencode]
---
**Symptom** — GPT models sometimes produced failing `task` tool calls around the optional `session_id` parameter (filling it when they meant a fresh task, or with invalid values).

**Root cause** — The parameter was named for the harness's internal concept (session) rather than for the model's purpose (resume a previous task), and its description did not say when to leave it empty.

**Fix · [[opencode]]** — `64e2bf8bf0` 2026-02-05 "adjust task tool description/input to alleviate tool call failures that sometimes occured w/ gpt models": renamed to `task_id` with "This should only be set if you mean to resume a previous task…" (`packages/opencode/src/tool/task.ts:47-50`); output now carries the `task_id` for resuming plus `<task_result>`.

**Lesson** — Name optional parameters by what the model should use them for, in the tool's own vocabulary, and say when to omit them.

Related: [[task-owned-subagent]] · [[tool-description-design]] · [[opencode--task-owned-subagent|opencode]]
