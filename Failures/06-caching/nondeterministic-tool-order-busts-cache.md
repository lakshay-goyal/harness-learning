---
type: failure
concepts: [cache-stable-prompt-prefix, cache-breakpoint-placement, mcp-integration]
harnesses: [opencode, codex]
---
**Symptom** — Prompt cache missed between consecutive requests of the same session although no tool was added or removed: the tool declarations block came out in a different order.

**Root cause** — The tool map is rebuilt every step from built-ins, MCP clients and plugins and was serialized in object insertion order, so any variation in assembly order changed the bytes at the top of the prompt (inference; the commit has no description).

**Fix · [[opencode]]** — `83bb216486` 2026-05-08 (#26370) "ensure tools are always in same order": tools sorted by name before `streamText` (then `packages/opencode/src/session/llm.ts`; HEAD `packages/opencode/src/session/llm/request.ts:184` `toSorted(([a],[b]) => a.localeCompare(b))`). v2 has no equivalent sort in `packages/core/src/tool/` (grep finds none), so its tool order is registry order (exposure unverified).

**Lesson** — Canonicalize every collection that lands in the cached prefix (tools, instruction files, MCP servers) at the provider boundary; insertion order is not stable across steps.

Related: [[cache-stable-prompt-prefix]] · [[volatile-system-prompt-prefix]] · [[late-tool-change-rewrites-cache]] · [[opencode--cache-stable-prompt-prefix|opencode impl]]

## Also: [[codex]] (folded from `nondeterministic-tool-order-breaks-cache`)
**Symptom** — MCP servers were stored in a `HashMap`, so the order of tools in the request changed across turns, "effectively breaking prompt caching in multi-turn sessions" (`ee2ccb5cb6` body, issue #2610).

**Root cause** — Iteration order of the hash map leaked into the serialized tool list, which sits at the head of the cached prefix.

**Fix · [[codex]]**
- `ee2ccb5cb6` 2025-08-25 (#2611), "Fix cache hit rate by making MCP tools order deterministic": sort the tools by name.
- The pattern persists: code-mode nested tool definitions are sorted and deduplicated by name (`codex-rs/tools/src/code_mode.rs:117-118`).
- Prefix items get deterministic UUIDv5 ids (`codex-rs/core/src/client.rs:908-940`).

**Lesson** — Anything serialized into the prompt prefix must have a deterministic order and deterministic ids. Hash-map iteration order is a cache bug.

Related: [[cache-stable-prompt-prefix]] · [[mcp-integration]] · [[late-tool-change-rewrites-cache]] · [[volatile-system-prompt-prefix]] · [[codex--cache-stable-prompt-prefix|codex]]
