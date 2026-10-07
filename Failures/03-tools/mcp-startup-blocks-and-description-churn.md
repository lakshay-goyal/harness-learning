---
type: failure
concepts: [mcp-integration, deferred-tool-loading, cache-stable-prompt-prefix]
harnesses: [pi, opencode]
---
**Symptom** — The first prompt blocked waiting for slow MCP servers to connect; and the codemode/tool_search descriptions **changed as servers connected** (listing tools, counts, server instructions), rewriting tool declarations and invalidating the prompt cache mid-session.

**Root cause** — Eager wait for all servers at session start; volatile server state embedded in tool descriptions.

**Fix · [[pi]]**
- `1c7e7df76` 2026-09-30 (refs #10212) — codemode description lists servers by name only, no tools/counts/instructions; scripts find tools with `searchTools()` / `describeNamespace()`.
- `e029c3ed0` 2026-09-30 — first prompt waits (≤ `DEFAULT_STARTUP_WAIT_MS = 10_000`) only for servers with `direct` tools; others connect in background; codemode scripts wait only for servers they name, tool_search/resource tools for all (`packages/coding-agent/src/extensions/mcp/index.ts:93,227-231,1122-1145`); codemode/tool_search activated from **config**, not connection state; servers listed in an `mcp_servers` system-prompt section whose changes are **appended** to the conversation (`docs/mcp.md:202`; CHANGELOG `:198`).

**Lesson** — Keep tool declarations byte-stable and independent of async connection state; put volatile facts in an appendable prompt section and wait lazily only for what a call needs.

Related: [[mcp-integration]] · [[deferred-tool-loading]] · [[cache-stable-prompt-prefix]] · [[transcript-carried-system-prompt]] · [[tool-description-design]] · [[pi--mcp-integration|pi]]

**Fix · [[pi]]** (caching angle, appended by 06-caching writer) — `tool_search` description deliberately "does not list the searchable tools or their namespaces, so it stays the same while tools are registered, for example when MCP servers connect" (`packages/coding-agent/src/extensions/tool-search/tool.ts:215-219`); CHANGELOG `:197` (#10212) "The `codemode` description no longer includes deferred tools, tool counts, or MCP server instructions, so it no longer changes when MCP servers connect". Section capped `MAX_SERVERS_SECTION_CHARS = 4096`, `MAX_SERVER_DESCRIPTION_CHARS = 250` (`extensions/mcp/index.ts:158,163`). See [[cache-stable-prompt-prefix]], [[pi--cache-stable-prompt-prefix]].

**Fix · [[opencode]]** — calibrating the startup timeout: `client.connect()` had no timeout, so a dead server blocked startup → `586e7347bd` 2026-01-03 5 s connect timeout → `e5abe1e78b` 2026-01-04 "bump default to 30 seconds (lots of people complained about 5…)" (npx/uvx cold starts) (`packages/opencode/src/mcp/index.ts:38`); per-call `mcp_timeout` `ed4ce67cdc` 2025-12-30, per server `dd1f981d23` 2026-01-15, catalog requests `f43b0d3afd` 2026-06-10, `resetTimeoutOnProgress` `a98d5732c0` 2026-06-16. Doc drift: config schema still says "Defaults to 5000 (5 seconds)" (`packages/core/src/v1/config/mcp.ts:21`). Description churn: server instructions live in a `<mcp_instructions>` system block, not tool descriptions (`packages/opencode/src/session/system.ts:121-137`). See [[opencode--mcp-integration]].
