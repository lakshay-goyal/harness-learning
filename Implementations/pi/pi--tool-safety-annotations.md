---
type: implementation
harness: pi
concept: tool-safety-annotations
commit: b30a6dd77
files: [packages/coding-agent/src/core/extensions/types.ts:518, packages/coding-agent/src/extensions/mcp/tools.ts:244, packages/coding-agent/src/extensions/mcp/resources.ts:198, packages/coding-agent/src/core/agent-session.ts:1502, packages/coding-agent/docs/extensions.md:166]
---
[[tool-safety-annotations]] in [[pi]].

## Mechanism
- `ToolAnnotations {readOnlyHint?, destructiveHint?, idempotentHint?, openWorldHint?}` — "Hints about what a tool does, with the meaning of MCP tool annotations. They come from the tool's author and are not verified; permission extensions can use them to decide which calls to confirm." (`packages/coding-agent/src/core/extensions/types.ts:518-531`). Optional field on `ToolDefinition` (`:610`) and on `pi.getAllTools()` info (`:2095`).
- Exposure: `getAllTools()` copies annotations per tool (`packages/coding-agent/src/core/agent-session.ts:1502`), alongside `exposure` and `namespace` (`packages/coding-agent/docs/extensions.md:166`).
- MCP tools: only boolean hints copied from the server (`ANNOTATION_HINTS`, `packages/coding-agent/src/extensions/mcp/tools.ts:244-253`), attached at `:269,279`. MCP resource tools (list/read/templates) hard-coded `readOnlyHint:true` (`packages/coding-agent/src/extensions/mcp/resources.ts:198,261,285,308`).
- Built-in tools (`read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`, `powershell`) declare **no** annotations (grep of `packages/coding-agent/src/core/tools` finds none) → treated by MCP defaults as non-read-only, possibly destructive, open-world.
- Defaults documented: "Missing hints take the MCP defaults: a tool is not read-only, and may be destructive and reach an open world." (`docs/extensions.md:166`).
- Consumer = user policy only; docs snippet (`docs/extensions.md:168-177`): `needsApproval = destructiveHint===true || (!readOnlyHint && ((destructiveHint ?? true) || (openWorldHint ?? true)))` then `ctx.ui.confirm` → `{block:true}` — "confirms the calls Codex asks approval for". MCP doc reiterates (`packages/coding-agent/docs/mcp.md:254`).

## Constants
| name | value | path:line |
|---|---|---|
| `ANNOTATION_HINTS` | readOnly/destructive/idempotent/openWorld | packages/coding-agent/src/extensions/mcp/tools.ts:244 |
| resource-tool hint | `{readOnlyHint:true}` | packages/coding-agent/src/extensions/mcp/resources.ts:198 |

## Evolution
- Before 2026-09-29: policy examples keyed on tool name + regex (`permission-gate.ts`).
- 2026-09-29 `8562bcf66` (v0.99.0, codemode + MCP, Armin Ronacher): annotations introduced with MCP integration (`git log -S readOnlyHint` → only this commit).

## Evidence commits
`8562bcf66`

## Quirks
- With built-ins unannotated, the docs' Codex-like policy prompts for *every* built-in call including `read` (inferred from defaults).
- Hints are author-asserted; a malicious MCP server can mark a destructive tool read-only.

## Failures
- none recorded
