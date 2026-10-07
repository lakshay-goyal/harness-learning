---
type: implementation
harness: opencode
concept: task-list-tool
commit: ecc4916b5a
files: [packages/opencode/src/tool/todo.ts:1-46, packages/opencode/src/tool/todowrite.txt:1-44, packages/opencode/src/agent/agent.ts:188, packages/opencode/src/agent/subagent-permissions.ts:18-26, packages/opencode/src/tool/task.ts:143-146, packages/opencode/src/session/prompt/anthropic.txt:96]
---
[[task-list-tool]] in [[opencode]].

## Mechanism
- `todowrite` takes the full list every call, persists it via `todo.update` and echoes it as JSON; result title = count of non-completed items, `N todos` (`packages/opencode/src/tool/todo.ts:31-37`).
- No read tool: `todoread` was dropped from the registry and deleted (see Evolution); the echoed list is the only read path.
- Description rules (`packages/opencode/src/tool/todowrite.txt`, 44 lines): "3+ distinct steps", exactly one `in_progress`, "Mark `completed` only after the required work is actually done, including any required verification", "When in doubt, use it."
- Scoping: `general` agent denies `todowrite` (`packages/opencode/src/agent/agent.ts:188`); child sessions get `todowrite: deny` unless the subagent's own ruleset mentions it (`packages/opencode/src/agent/subagent-permissions.ts:18-26`; `packages/opencode/src/tool/task.ts:143-146`) → [[permission-ruleset]].
- Prompt cadence per family: `anthropic.txt:96` "IMPORTANT: Always use the TodoWrite tool…"; `default.txt` (non-Claude fallback) carries no todo instructions.
- v2 runtime: `TodoWriteTool` is in the core built-in list (`packages/core/src/tool/builtins.ts:43`).

## Constants
| name | value | path:line |
|---|---|---|
| description size | 8845 → 2012 bytes (`548648a3d9`) | `packages/opencode/src/tool/todowrite.txt` |

## Evolution
- 2025-07-29 `df03e182d2` strip todo instructions from non-Anthropic prompts.
- 2025-08-07 `8750744068` re-enable todo tool; 2025-08-12 `5cc44c872e` disabled for qwen; 2025-09-08 `1cea8b9e77` re-enabled.
- 2025-10-25 `795b845782` Claude Code sync drops "Always use TodoWrite"; 2025-10-28 `22821744ef` re-adds it.
- 2026-01-19 `3515b4ff7d` omitted for openai models; 2026-01-21 `d9f0287d74` added back.
- 2026-02-02 `1275c71a63` `todoread` commented out of the registry; 2026-03-25 `77fc88c8ad` dead code deleted.
- 2026-05-16 `548648a3d9` 211-line Claude Code description → 44 lines.

## Quirks / drift
- `general` subagent (no own prompt) receives `anthropic.txt:96` "Always use the TodoWrite tool" while its permission denies `todowrite` → [[prompt-names-unavailable-tools]].

Contrast: pi has no todo tool → [[no-todo-tool]].
