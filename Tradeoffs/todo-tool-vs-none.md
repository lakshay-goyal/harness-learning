---
type: tradeoff
concepts: [task-list-tool, branch-scoped-extension-state, plugin-tools]
harnesses: [pi, opencode]
---
# todo-tool-vs-none

**Axis**: does the harness give the model a structured todo-list tool, or let it keep plans in files?

| option | pi | opencode | evidence |
|---|---|---|---|
| No todo tool; TODO.md with checkboxes | ✅ | — | pi: "to-do lists generally confuse models more than they help … make it stateful by writing to a file: TODO.md" (`9066f58ca`) → [[no-todo-tool]] |
| Extension tool, state in tool-result `details` (branch-safe) | ✅ `examples/extensions/todo.ts` | — | `todo.ts:1-11` → [[branch-scoped-extension-state]] |
| Built-in write-only list tool | — | ✅ `todowrite`: model sends the full list each call; persisted in `TodoTable`; rendered | `packages/opencode/src/tool/todo.ts:1-46` → [[task-list-tool]] |
| Read tool for the list | — | removed (`todoread` deleted `77fc88c8ad`) | → [[removed-builtin-tools]] |
| Per-model toggling | — | disabled for qwen `5cc44c872e` → re-enabled `1cea8b9e77`; omitted for `gpt-` `3515b4ff7d` → re-added `d9f0287d74` | features A12 |
| Subagents | — | denied to `general` and to children by default | `packages/opencode/src/agent/agent.ts:182-193`; `packages/opencode/src/tool/task.ts:143-146` |

**Cost of having none**: models trained with a todo tool still try to call it. pi's Codex bridge had to say "UPDATE_PLAN DOES NOT EXIST — NEVER use: … todowrite, todoread" (`1650041a6` → dropped `6484ae279`).

**When each wins**
- **File (pi)**: the plan must survive compaction verbatim and be human-editable; branching sessions need state tied to the tree, not to harness memory.
- **Tool (opencode)**: models trained on TodoWrite/update_plan behave better when the tool exists; the UI can render progress. The flip-flopping per model family shows the benefit is model-dependent, and "todos confuse models" (pi) is equally anecdotal (no eval on either side).

Related: [[no-todo-tool]] · [[plan-mode-vs-none]] · [[minimal-vs-rich-toolset]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
