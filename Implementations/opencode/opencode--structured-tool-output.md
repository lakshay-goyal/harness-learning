---
type: implementation
harness: opencode
concept: structured-tool-output
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:82, packages/opencode/src/session/prompt.ts:1244-1289, packages/opencode/src/session/prompt.ts:1565-1591, packages/opencode/src/tool/edit.ts:190-199, packages/core/src/tool/tool.ts:71-135, packages/opencode/src/mcp/catalog.ts:68-79]
---
[[structured-tool-output]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Terminating submit tool**: when the user message has `format.type === "json_schema"`, a `StructuredOutput` tool with the user schema is added, `STRUCTURED_OUTPUT_SYSTEM_PROMPT` ("You MUST use the StructuredOutput tool…") is appended, and `toolChoice: "required"` is sent every step (`packages/opencode/src/session/prompt.ts:82,1244-1289`).
- The SDK validates args; `execute` stores the value in a closure; the loop breaks and stores `message.structured`; model sees "Structured output captured successfully." (`prompt.ts:1565-1591`) → [[constrained-tool-sampling]].
- Every tool returns `{title, output, metadata}`: `metadata` (diff, filediff, diagnostics, truncation `outputPath`) drives UI only (`packages/opencode/src/tool/edit.ts:190-199`).
- Success stated in text, never implied: "Edit applied successfully.", "Wrote file successfully." → [[empty-success-output-read-as-failure]].
- MCP: when `content` is non-empty it wins over `structuredContent`; structured JSON is used only when content is empty (`packages/opencode/src/mcp/catalog.ts:68-79`).
### v2 runtime
- `Tool.make({description, input, output, structured?, toStructuredOutput?, execute, toModelOutput?})`: decode input → execute → encode output → pure `toModelOutput` projection; output JSON Schema advertised (`packages/core/src/tool/tool.ts:71-135`).
- Laws: single executor, codec boundary, durable identity, scoped registration, captured execution, stale rejection, storage encapsulation (`specs/v2/tools.md:172-180`).

## Constants
| name | value | path:line |
|---|---|---|
| structured-output `toolChoice` | `"required"` | `packages/opencode/src/session/prompt.ts:1285` |

## Evolution
- 2026-01-12 `66f9bdab32` explicit success lines in edit/write.
- 2026-06-06 `660a00d317` v2 unified tool architecture.
- 2026-06-29 `fd213e6df6` MCP: prefer content over structured output.

## Quirks / drift
- None noted.

Contrast: pi sends `content` + `structuredContent` with an outputSchema and different limits for scripts vs model → [[pi--structured-tool-output|pi]].
