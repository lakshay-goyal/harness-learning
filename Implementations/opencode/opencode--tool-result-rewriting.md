---
type: implementation
harness: opencode
concept: tool-result-rewriting
commit: ecc4916b5a
files: [packages/opencode/src/session/tools.ts:100-128, packages/opencode/src/session/tools.ts:176-215, packages/opencode/src/tool/code-mode.ts:142-185]
---
[[tool-result-rewriting]] in [[opencode]].

## Mechanism
### Legacy runtime
- Each tool's AI SDK `execute`: `plugin.trigger("tool.execute.before", {tool, sessionID, callID}, {args})` (args mutable) → `item.execute(args, ctx)` → attachments get ids → `plugin.trigger("tool.execute.after", {tool, sessionID, callID, args}, output)` where `output = {title, output, metadata, …}` is mutable in place (`packages/opencode/src/session/tools.ts:100-128`).
- Same pair wraps MCP tools (`tools.ts:176-215`) and each nested call inside code mode (`packages/opencode/src/tool/code-mode.ts:142-185`).
- Semantics: hooks run in plugin order on one shared mutable object (chained by mutation, not field patches); `Plugin.trigger` runs hooks sequentially via `Effect.promise` with no catch, so a throwing hook becomes a defect that aborts the tool execution (`packages/opencode/src/plugin/index.ts:284-297`).
- Separate `tool.definition` hook rewrites the declaration, not results → [[tool-description-design]].
### v2 runtime
- `toModelOutput` is a pure projection inside `Tool.make` (`packages/core/src/tool/tool.ts:71-132`); no after-hook equivalent found (unverified).

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- 2025-10-23 `3c7b229d8b` `tool.execute.after` allowed to modify MCP output (previously ignored); 2025-10-30 `149f5eaa2e` MCP metadata preserved in `tool.execute.after`.

## Quirks / drift
- None noted.

Contrast: pi applies field-level patches chained across `afterToolCall` handlers, and hook throws become error results → [[pi--tool-result-rewriting|pi]].
