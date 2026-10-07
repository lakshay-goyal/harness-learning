---
type: concept
stage: tool-design
tier: candidate
aliases: [sanitize_json_schema, compact_large_tool_schema, DEFAULT_COMPACT_TOOL_SCHEMA_BYTES, tool_input_schema_max_bytes, parse_tool_input_schema, schema compaction, foreign schema sanitizer]
harnesses: [codex]
---
Rewrite foreign tool schemas (MCP servers, client-supplied dynamic tools, plugins) into the harness's supported schema subset, prune unreachable definitions, and shrink oversized schemas in increasingly lossy passes before declaring them to the model.

## Why
- Foreign schemas use keywords the harness/provider can't represent; a naive sanitizer that fills gaps with a concrete type **invents constraints** that steer arguments wrong ([[invented-schema-constraint]]).
- Kept numeric/length bounds or unsupported keywords make strict providers reject the whole request ([[strict-tool-schema-rejections]]).
- One giant MCP schema can eat thousands of prompt tokens on every request; a budget is needed, but blunt compaction strips field guidance the model needs (codex `9fe55d68e6`).

## Design space
- Pass schema through untouched vs **sanitize into supported subset** (codex `sanitize_json_schema`) vs per-provider strict transformation ([[constrained-tool-sampling]], pi).
- Unknown/untyped nodes: coerce to a scalar (codex before `4dbca61e20`) vs **widen to `{}`** (codex now). Rule: widen, never narrow.
- Bounds (`minimum`/`maximum`/`maxLength`): keep vs **drop** (codex `ac644ed112`).
- Size budget: none vs **best-effort lossy passes** (descriptions → definitions → deep objects → compositions; codex 5000 B, depth 3) vs hard fallback to an open object (codex agent-plugin MCP > 8000 B).
- Budget scope: per tool, per server override, total per plugin (codex 64000 B), separate budget for code-mode TypeScript rendering → [[code-mode]].
- Trusted first-party schemas exempt from compaction (codex web search, `parse_tool_input_schema_without_compaction`).

## Implementations
- [[codex--tool-schema-normalization|codex]] — `codex-rs/tools/src/json_schema.rs` sanitizer + `json_schema/compaction.rs` 4-pass compactor; MCP / agent-plugin byte caps; configurable per server and for code mode.

## Failures
- [[invented-schema-constraint]]
- (02) [[strict-tool-schema-rejections]]

## Related
[[constrained-tool-sampling]] · [[mcp-integration]] · [[client-supplied-dynamic-tools]] · [[tool-wire-kinds]] · [[tool-description-design]] · [[code-mode]] · [[deferred-tool-loading]]
