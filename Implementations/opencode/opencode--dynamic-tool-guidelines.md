---
type: implementation
harness: opencode
concept: dynamic-tool-guidelines
commit: ecc4916b5a
files: [origin/v2:packages/core/src/session/system-prompt.ts:5-28, origin/v2:packages/core/src/session/runner/prompt/system.txt:7, origin/v2:packages/core/src/plugin/optimize.ts:78-86, packages/opencode/src/session/system.ts:107-137, packages/opencode/src/tool/registry.ts:277]
---
[[dynamic-tool-guidelines]] in [[opencode]]. Present only on origin/v2; legacy hard-codes tool advice in each family prompt.

## Mechanism
### Legacy runtime
- Tool advice is static prose in each family prompt (e.g. anthropic.txt "use the Task tool instead of running search commands directly", gpt.txt "Always use apply_patch") and in the shell description's tool mapping; it does not follow the actual toolset → [[prompt-names-unavailable-tools]].
- Conditional blocks that do follow state: skills catalog omitted when `skill` is denied; MCP server instructions dropped when all of the server's tools are denied (`packages/opencode/src/session/system.ts:107-137`); task description lists only permitted subagents (`packages/opencode/src/tool/registry.ts:277`).
### origin/v2 branch
- `SessionSystemPrompt.render(prompt, tools)` fills `${OPENCODE_TOOL_GUIDANCE}` (`origin/v2:packages/core/src/session/runner/prompt/system.txt:7`) with lines only for tools present (`origin/v2:packages/core/src/session/system-prompt.ts:9-27`):
  - `shell` → "Prefer dedicated tools over shell commands; fall back to the shell when a tool cannot do what you need." + "Do not chain shell commands with separators like `echo \"====\";`…"
  - `write` → "Use the write tool to create files or completely replace their content. Prefer using the edit tool for targeted changes."
  - `edit` → `oldString` exactness/uniqueness/`replaceAll` rules.
- Family plugins curate tools *before* rendering ("Curate tools before rendering their guidance, including for agents with a custom system prompt"), then re-render their template with the same tool list (`origin/v2:packages/core/src/plugin/optimize.ts:78-86`) → [[model-specific-toolset]].

## Constants
| name | value | path:line |
|---|---|---|
| guided tools | shell, write, edit | `origin/v2:packages/core/src/session/system-prompt.ts:11-25` |

## Evolution
- 2026-08-14 `8640ea3374` "minimize system prompt" introduces the placeholder (origin/v2).

## Quirks / drift
- Guidance is centralized in one function keyed on tool names, not owned per tool.

Contrast: pi lets each tool contribute `promptSnippet`/`promptGuidelines`, recomputed per request from the declared set → [[pi--dynamic-tool-guidelines|pi]].
