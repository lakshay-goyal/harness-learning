---
type: failure
concepts: [parallel-tool-execution, mcp-integration, tool-safety-annotations]
harnesses: [codex]
---
**Symptom** — MCP tool calls always ran serially (exclusive lock) unless the server config opted in, even for tools that declare themselves read-only — parallel batches of MCP reads crawled.

**Root cause** — The per-tool parallel flag for MCP came only from user config; the MCP spec's own `readOnlyHint` was ignored as a concurrency signal. Conversely, a cached (possibly stale) catalog could claim read-only for a tool that changed.

**Fix · [[codex]]**
- `c83ba22359` 2026-05-21 "Allow parallel MCP tool calls when annotated readOnly (#23750)" — parallel iff `readOnlyHint == true` or server `supports_parallel_tool_calls` (`codex-rs/core/src/tools/handlers/mcp.rs:142-150`, tests `:880-915`).
- `3bbf1fe757` 2026-07-27 — cached tool definitions exposed before live startup have the read-only hint cleared (stale cache must not grant concurrency).
- Approval side uses the same annotations with pessimistic defaults ([[mcp-annotation-defaults-unsafe]]).

**Lesson** — Use MCP safety annotations as the concurrency signal, but distrust them when they come from a stale cache.

Related: [[parallel-tool-execution]] · [[mcp-integration]] · [[tool-safety-annotations]] · [[codex--parallel-tool-execution|codex]] · [[codex--mcp-integration|codex mcp]]
