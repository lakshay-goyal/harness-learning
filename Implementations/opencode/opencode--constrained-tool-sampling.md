---
type: implementation
harness: opencode
concept: constrained-tool-sampling
commit: ecc4916b5a
files: [packages/opencode/src/session/llm/request.ts:148-158, packages/opencode/src/session/prompt.ts:82, packages/opencode/src/session/prompt.ts:1244-1285, packages/llm/src/protocols/openai-responses.ts:259-266]
---
[[constrained-tool-sampling]] in [[opencode]].

## Mechanism
- **Strict mode deliberately off**: OpenAI / Azure / Bedrock-mantle (Responses family) get `strict: false` on every function tool — "Codex parity … so MCP-sourced and dynamic schemas that don't satisfy OpenAI's structured-outputs constraints still register" (`packages/opencode/src/session/llm/request.ts:148-158`). v2 Responses lowering hardcodes `strict: false` with a TODO to make it opt-in (`packages/llm/src/protocols/openai-responses.ts:259-266`).
- Schema compatibility is handled by rewriting instead → [[opencode--tool-schema-lowering|tool-schema-lowering]].
- **Structured output** (user message `format: json_schema`): adds a `StructuredOutput` tool, pushes `STRUCTURED_OUTPUT_SYSTEM_PROMPT` ("You MUST use the StructuredOutput tool … Do NOT respond with plain text"), and sets `toolChoice: "required"` (`packages/opencode/src/session/prompt.ts:82,1244,1271,1285`). Text-only ending → `StructuredOutputError("Model did not produce structured output")`.

## Constants
| name | value | path:line |
|---|---|---|
| Responses-family `strict` | `false` | `packages/opencode/src/session/llm/request.ts:157` |

## Evolution
- 2026-06-16 `213ff3f2d7` OpenAI MCP tool schemas sanitized (Codex lowering port) alongside `strict: false`.

Failures: [[strict-tool-schema-rejections]].

Contrast: [[pi--constrained-tool-sampling|pi]] opts into provider strict sampling (`makeStrictJsonSchema`, grammars); opencode turns strict off and lowers schemas instead.
