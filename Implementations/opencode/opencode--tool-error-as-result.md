---
type: implementation
harness: opencode
concept: tool-error-as-result
commit: ecc4916b5a
files: [packages/opencode/src/tool/invalid.ts:9-21, packages/opencode/src/tool/tool.ts:120-129, packages/opencode/src/tool/edit.ts:723-728, packages/opencode/src/mcp/catalog.ts:68-79, packages/core/src/v1/permission.ts:9-25, packages/core/src/session/runner/llm.ts:254, packages/core/src/session/runner/llm.ts:301, packages/core/src/session/runner/llm.ts:323, packages/llm/src/tool-runtime.ts:23-76]
---
[[tool-error-as-result]] in [[opencode]].

## Mechanism
### Legacy runtime
- Tool execute dies (`Effect.orDie`) → AI SDK `tool-error` → part `status: "error"` → replayed to the model as error output.
- Unknown/misnamed tool → `invalid` sink result → [[tool-argument-repair]].
- One error string per remedy in `edit`: not found vs multiple matches vs disproportionate match (`packages/opencode/src/tool/edit.ts:709-728`) → [[ambiguous-tool-error-causes-retry-loop]].
- Permission outcomes become model-facing text: "The user rejected permission to use this specific tool call[ with the following feedback: …]" / "The user has specified a rule which prevents you from using this specific tool call…" (`packages/core/src/v1/permission.ts:9-25`) → [[permission-ruleset]].
- MCP `isError: true` results mapped to failures; human-readable `content` preferred over `structuredContent` (`packages/opencode/src/mcp/catalog.ts:68-79`) → [[mcp-error-result-treated-as-success]].
### v2 runtime
- Unexpected tool defect → every unsettled call failed with `Tool execution failed: <message>`, loop continues (`packages/core/src/session/runner/llm.ts:323`); provider-hosted call without result → "Provider did not return a tool result" (`:301`); calls after the step limit → "Tools are disabled after the maximum agent steps" (`:254`).
- Leaf tools map typed permission errors (`CorrectedError`, `BlockedError`) to generic `Unable to …` strings, so the model loses the user's correction text (inference from code; no test found) → [[permission-feedback-lost-in-tool-error]].

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- 2025-07-07 `0d50c867ff` MCP raw content arrays corrupted sessions.
- 2025-08-04 `0a42068fbb` tool call errors returned to model.
- 2025-09-05 `900fe5ca04` edit error split.
- 2026-06-14 `dfb616f067` MCP `isError` handled; 2026-06-29 `fd213e6df6` prefer content over structured output.
- 2026-06-23 `c17b9557f1` structured error messages preserved (were `[object Object]`-like).
- 2026-06-27 `93159bccbf` v2: return unexpected local tool defects to the model and continue.

## Quirks / drift
- Dismissing a `question` is not an error result: it raises `RejectedError` and stops the loop → [[ask-user-tool]].

Contrast: pi tools throw and the loop renders the message as an `isError` result → [[pi--tool-error-as-result|pi]].
