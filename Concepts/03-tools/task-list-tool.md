---
type: concept
stage: tools
tier: candidate
aliases: [todowrite, todoread, TodoWrite, Todo.Service, todo list]
harnesses: [opencode]
---
A tool through which the model maintains a structured todo list that the harness persists and renders.

## Why
- Long multi-step tasks drift: without an explicit list the model forgets steps or declares completion early.
- The user sees progress in the UI instead of parsing prose.
- The list survives compaction as tool-call history, while prose plans may be summarized away.

## Design space
- **Write-only full-list replace, echoed back as the result** (opencode, after `todoread` was removed) vs separate read tool vs item-level patch ops.
- States: pending / in_progress / completed / cancelled with exactly one in_progress (opencode).
- Cadence driven by prompt text ("3+ distinct steps", "When in doubt, use it") vs harness reminders.
- Per-model gating: hide for models that misuse it (opencode toggled for qwen and GPT, both reverted) → [[todo-tool-usage-calibration]].
- Scope: per session; denied to subagents by default (opencode).
- Absent; model keeps a plan in prose or a file (pi, [[no-todo-tool]]).

## Implementations
- [[opencode--task-list-tool|opencode]] — `todowrite` persists the full list via `Todo.update`, title `N todos`; `general` agent and child sessions deny it unless allowed.

## Failures
- [[todo-tool-usage-calibration]]
- [[prompt-names-unavailable-tools]]

## Tradeoffs
- [[todo-tool-vs-none]]

## Related
[[no-todo-tool]] · [[per-model-system-prompt]] · [[model-specific-toolset]] · [[tool-description-design]] · [[task-owned-subagent]]
