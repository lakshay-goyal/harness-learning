---
type: failure
concepts: [cache-stable-prompt-prefix, cache-breakpoint-placement]
harnesses: [opencode]
---
**Symptom** — Prompt cache missed between consecutive requests of the same session although no tool was added or removed: the tool declarations block came out in a different order.

**Root cause** — The tool map is rebuilt every step from built-ins, MCP clients and plugins and was serialized in object insertion order, so any variation in assembly order changed the bytes at the top of the prompt (inference; the commit has no description).

**Fix · [[opencode]]** — `83bb216486` 2026-05-08 (#26370) "ensure tools are always in same order": tools sorted by name before `streamText` (then `packages/opencode/src/session/llm.ts`; HEAD `packages/opencode/src/session/llm/request.ts:184` `toSorted(([a],[b]) => a.localeCompare(b))`). v2 has no equivalent sort in `packages/core/src/tool/` (grep finds none), so its tool order is registry order (exposure unverified).

**Lesson** — Canonicalize every collection that lands in the cached prefix (tools, instruction files, MCP servers) at the provider boundary; insertion order is not stable across steps.

Related: [[cache-stable-prompt-prefix]] · [[volatile-system-prompt-prefix]] · [[late-tool-change-rewrites-cache]] · [[opencode--cache-stable-prompt-prefix|opencode impl]]
