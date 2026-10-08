---
type: implementation
harness: codex
concept: constrained-tool-sampling
commit: 622e9e3696
files: [codex-rs/core/assets/tools/apply_patch.lark, codex-rs/core/src/tools/handlers/apply_patch_spec.rs:5-27, codex-rs/core/src/tools/code_mode/execute_spec.rs:20-43, codex-rs/tools/src/responses_api.rs:16-40, codex-rs/core/src/tools/handlers/extension_tools.rs:269, codex-rs/core/src/client.rs:973-977, codex-rs/core/src/client_common.rs:39-52, codex-rs/tools/src/json_schema.rs:71-206]
---
[[constrained-tool-sampling]] in [[codex]]. The wire shapes (function/custom/namespace) are in [[tool-wire-kinds]]; this note covers decoding constraints.

## Mechanism
- **Grammar-constrained freeform tools (Lark).**
  - `apply_patch` is a Responses `custom` tool: `format: {type: "grammar", syntax: "lark", definition}`. The grammar `codex-rs/core/assets/tools/apply_patch.lark` (19 lines) is compiled in with `include_str!`. The description reads "This is a FREEFORM tool, so do not wrap the patch in JSON." (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:5-27`).
  - In multi-environment turns the grammar is patched by string replacement of its `start:` rule, adding an optional `*** Environment ID: ` line (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:10-15`).
  - Code-mode `exec` is also a Lark freeform tool (`codex-rs/core/src/tools/code_mode/execute_spec.rs:20-43`).
  - The model's raw text arrives as the `CustomToolCall` input. There is no JSON layer, and handlers match `ToolPayload::Custom`.
- **Freeform only, since 2026-05-08.** The catalog field `apply_patch_tool_type` once chose freeform (grammar) or function (JSON) per model ([[codex--model-catalog|codex catalog]]).
  - Bedrock-served GPT models were briefly forced to the function variant (`0db6811b7c` 2026-04-24; [[endpoint-rejects-request-field]]).
  - `e783341b70` 2026-05-08 deleted `ApplyPatchToolType::Function`, the JSON spec and the function-payload handler: "Keeping the old JSON/function-style registration and parsing path left another way for models and tests to invoke `apply_patch`". Freeform is now the only path, "including Bedrock catalog metadata".
- **JSON-schema strict.**
  - `ResponsesApiTool.strict` exists with a "TODO: Validation" comment (`codex-rs/tools/src/responses_api.rs:33-40`).
  - Extension tools set `strict: true` (`codex-rs/core/src/tools/handlers/extension_tools.rs:269`).
  - MCP and dynamic tools go through `sanitize_json_schema`, which degrades gracefully to `strict: false` registration ([[tool-schema-lowering]]; `codex-rs/tools/src/json_schema.rs:71-206`).
- **Structured final output.** The `--output-schema` path sends `text.format` with `output_schema_strict` (default true) (`codex-rs/core/src/client.rs:973-977`; `codex-rs/core/src/client_common.rs:39-52`).

## Evolution
- `6fcc528a43` 2025-06-03: lenient parse of JSON-wrapped `apply_patch` (GPT-4.1 produced predictably invalid invocations).
- `236c4f76a6` 2025-08-22: freeform Lark `apply_patch` introduced. Two days later `4157788310` 2025-08-24 noted "issues… let's disable by default until it stabilizes".
- `6f75114695` 2025-09-02: "[apply-patch] Fix lark grammar".
- `4764fc1ee7` 2025-10-04: freeform tool plus Lark grammar so the sampler is constrained ([[malformed-tool-json-crashes]]).
- `42e22f3bde` 2026-02-11: `js_repl` freeform tool. `73fd939296` 2026-02-20: grammar blocks for wrapped payload prefixes. Removed in `8a559e7938` 2026-04-24.
- `e783341b70` 2026-05-08 (#21651): "Delete function-style apply_patch". The function/JSON variant was removed for all providers, Bedrock included.
- `4dbca61e20` 2026-05-18: untyped schema nodes become `{}` instead of an invented `string`.
- `ac644ed112` 2026-08-26: stop preserving `minimum`/`maximum`/`maxLength` bounds.

## Versus pi
- [[pi--constrained-tool-sampling|pi]] strictifies JSON schemas per provider (`prefer`/`require`), and its grammar bridge re-encodes grammar output into JSON deltas.
- Codex uses grammar-constrained freeform text as the *only* format for its most important tool (the patch), with no JSON fallback since `e783341b70`. JSON strict mode is secondary.
- See [[edit-tool-variants]].

## Failures
- [[grammar-constrained-tool-instability]]
- [[malformed-tool-json-crashes]]
- [[strict-tool-schema-rejections]]
- [[endpoint-rejects-request-field]]
