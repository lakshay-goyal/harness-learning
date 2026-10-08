---
type: failure
concepts: [tool-safety-annotations, mcp-integration, tool-call-gate]
harnesses: [codex]
---
**Symptom** — MCP tools marked `destructiveHint` did not force an approval when also `readOnlyHint` (`d3cf8bd0fa`), and tools with *no* annotations were treated permissively.

**Root cause** — Hints were combined optimistically and missing hints were read as "safe", the opposite of the MCP spec defaults.

**Fix · [[codex]]** — 2026-02-20 `d3cf8bd0fa` "require approval for destructive MCP tool calls (#12353)" — destructive short-circuits; 2026-03-25 `32c4993c8a` "default approval behavior for mcp missing annotations (#15519)" — unannotated tools default to spec values `readOnlyHint=false`, `destructiveHint=true`, `openWorldHint=true`. Current logic `codex-rs/core/src/mcp_tool_call.rs:2461-2478`; connectors same defaults (`codex-rs/connectors/src/app_tool_policy.rs:235-236`).

**Lesson** — Missing safety metadata must default to the spec's pessimistic values, and "destructive" must dominate any other hint.

Related: [[tool-safety-annotations]] · [[mcp-integration]] · [[tool-call-gate]] · [[codex--tool-safety-annotations|codex]] · [[unannotated-mcp-tools-serialized]]
