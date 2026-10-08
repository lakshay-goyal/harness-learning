---
type: failure
concepts: [tool-schema-lowering, mcp-integration]
harnesses: [codex]
---
**Symptom** — Untyped nodes in MCP / dynamic tool schemas were filled in as `string` by the sanitizer, inventing "a scalar constraint that the provider did not specify" that "could incorrectly steer tool arguments away from the provider's actual accepted shape" (`4dbca61e20` body). Related: generated declarations kept `minimum`/`maximum`/`maxLength` bounds (`ac644ed112`), and size compaction stripped field guidance from the web-search tool schema (`9fe55d68e6`).

**Root cause** — Normalizing foreign schemas into a smaller supported subset by *narrowing* (choosing a concrete type) instead of *widening*.

**Fix · [[codex]]**
- `4dbca61e20` 2026-05-18 "default unknown tool schemas to empty schemas" — untyped nodes → `{}` (`codex-rs/tools/src/json_schema.rs:71-206`).
- `9fe55d68e6` 2026-05-27 trusted first-party schemas (standalone web search) bypass compaction (`parse_tool_input_schema_without_compaction`, `json_schema.rs:41-46`).
- `ac644ed112` 2026-08-26 "Stop preserving bounds in tool input schemas".
- `339e981ba7` 2026-09-24 compaction budgets configurable per MCP server / code mode.

**Lesson** — When normalizing foreign schemas, widen, never narrow; and don't let budget-driven compaction strip guidance from schemas you own.

Related: [[tool-schema-lowering]] · [[mcp-integration]] · [[strict-tool-schema-rejections]] · [[constrained-tool-sampling]] · [[codex--tool-schema-lowering|codex]]
