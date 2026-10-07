---
type: failure
concepts: [cache-stable-prompt-prefix, mcp-integration]
harnesses: [codex]
---
**Symptom** — MCP servers were stored in a `HashMap`, so the order of tools in the request changed across turns, "effectively breaking prompt caching in multi-turn sessions" (`ee2ccb5cb6` body, issue #2610).

**Root cause** — Iteration order of the hash map leaked into the serialized tool list, which sits at the head of the cached prefix.

**Fix · [[codex]]**
- `ee2ccb5cb6` 2025-08-25 (#2611), "Fix cache hit rate by making MCP tools order deterministic": sort the tools by name.
- The pattern persists: code-mode nested tool definitions are sorted and deduplicated by name (`codex-rs/tools/src/code_mode.rs:117-118`).
- Prefix items get deterministic UUIDv5 ids (`codex-rs/core/src/client.rs:908-940`).

**Lesson** — Anything serialized into the prompt prefix must have a deterministic order and deterministic ids. Hash-map iteration order is a cache bug.

Related: [[cache-stable-prompt-prefix]] · [[mcp-integration]] · [[late-tool-change-rewrites-cache]] · [[volatile-system-prompt-prefix]] · [[codex--cache-stable-prompt-prefix|codex]]
