---
type: implementation
harness: codex
concept: tool-schema-lowering
commit: 622e9e3696
files: [codex-rs/tools/src/json_schema.rs:21-206, codex-rs/tools/src/json_schema/compaction.rs:15-40, codex-rs/tools/src/responses_api.rs:14, codex-rs/tools/src/responses_api.rs:121-151, codex-rs/core/src/mcp_tool_exposure.rs:18-19, codex-rs/core/src/tools/spec_plan.rs:657-671]
---
[[tool-schema-lowering]] in [[codex]].

## Mechanism
- **Sanitizer** `sanitize_json_schema` (`codex-rs/tools/src/json_schema.rs:71-206`), doc list: ensures every typed object has `"type"` when required; preserves `anyOf`/`oneOf`/`allOf`; preserves `$ref` + reachable local `$defs`/`definitions`; collapses `const` → single-value `enum` (`:119-120`); fills required child fields for object/array (incl. nullable unions) with permissive defaults; object schemas with no recognized hints → `{}`. Unreachable definitions pruned (`prune_unreachable_definitions`, `:207-258`); malformed definition tables dropped so `strict: false` registration degrades gracefully. Keys: `DEFINITION_TABLE_KEYS = ["$defs","definitions"]`, `SCHEMA_CHILD_KEYS = ["items","anyOf","oneOf","allOf"]` (`:21-23`).
- **Compaction** `compact_large_tool_schema` — "best-effort rather than a hard cap: it runs only after schema sanitization/pruning and applies increasingly lossy passes while the schema remains over budget" (`codex-rs/tools/src/json_schema/compaction.rs:18-29`). Passes in order: `strip_schema_descriptions` → `drop_schema_definitions` (refs rewritten to `{}`) → `collapse_deep_schema_objects_from_root` (depth > 3) → `prune_schema_compositions` (`:33-38`). Budget measured on compact normalized JSON UTF-8 bytes.
- Entry points: `parse_tool_input_schema` (default budget), `parse_tool_input_schema_with_max_bytes`, `parse_tool_input_schema_without_compaction` for trusted schemas (`json_schema.rs:26-46`; `9fe55d68e6` "dont compact standalone websearch schema").
- MCP: `mcp_tool_to_responses_api_tool(…, schema_max_bytes)` uses per-server `tool_input_schema_max_bytes` when set (`codex-rs/tools/src/responses_api.rs:121-134`); agent-plugin MCP tools whose serialized spec > `MAX_SERIALIZED_MCP_TOOL_BYTES = 8_000` get an open object schema (`additionalProperties: true`) (`responses_api.rs:136-151`); agent-plugin MCP totals `MAX_AGENT_PLUGIN_MCP_SPEC_BYTES = 8_000` / `MAX_AGENT_PLUGIN_MCP_TOTAL_BYTES = 64_000` (`codex-rs/core/src/mcp_tool_exposure.rs:18-19`).
- Code mode renders schemas into TypeScript declarations under `code_mode.tool_input_schema_max_bytes` (`codex-rs/core/src/tools/spec_plan.rs:657-671`) → [[code-mode]].
- `ResponsesApiTool.strict` is not validated ("TODO: Validation", `responses_api.rs:33-40`) → [[tool-wire-kinds]].

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_COMPACT_TOOL_SCHEMA_BYTES` | 5_000 | `codex-rs/tools/src/json_schema/compaction.rs:15` |
| `MAX_COMPACT_TOOL_SCHEMA_DEPTH` | 3 | `codex-rs/tools/src/json_schema/compaction.rs:16` |
| `MAX_SERIALIZED_MCP_TOOL_BYTES` | 8_000 (agent-plugin MCP → open schema) | `codex-rs/tools/src/responses_api.rs:14` |
| `MAX_AGENT_PLUGIN_MCP_SPEC_BYTES` / `_TOTAL_BYTES` | 8_000 / 64_000 | `codex-rs/core/src/mcp_tool_exposure.rs:18-19` |

## Evolution
- 2026-05-18 `4dbca61e20` "default unknown tool schemas to empty schemas" — untyped nodes had become `string`, "invented a scalar constraint that the provider did not specify … could incorrectly steer tool arguments" ([[invented-schema-constraint]]).
- 2026-05-27 `9fe55d68e6` compaction (then ~4k-byte threshold per commit body) stripped field guidance from the standalone web-search tool → bypass for trusted schemas.
- 2026-08-26 `ac644ed112` "Stop preserving bounds in tool input schemas".
- 2026-09-24 `339e981ba7` "Make MCP and Code Mode input schema budgets configurable".

## Versus pi
pi's schema rewriting is provider-facing (strict-mode transformation with fallback, [[pi--constrained-tool-sampling]]); no size compaction of MCP schemas found in the pi notes (unverified; MCP default exposure is codemode-only, which sidesteps per-request schema cost).
