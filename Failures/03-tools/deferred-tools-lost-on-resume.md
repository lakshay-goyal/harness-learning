---
type: failure
concepts: [deferred-tool-loading, mcp-integration]
harnesses: [pi]
---
**Symptom** — MCP tools that `tool_search` had loaded **vanished after resume or `/reload`**; earlier, tools registered dynamically at runtime stayed invisible until `/reload`, and tool refreshes before the next request dropped prompt overrides.

**Root cause** — The tool loadout is persisted state with asynchronous dependencies: the session restored its active set before MCP servers reconnected, so not-yet-registered tools were dropped.

**Fix · [[pi]]**
- `bc2fa8d6d` 2026-03-02 (#1720) — dynamic tool registration refresh (`packages/coding-agent/src/core/agent-session.ts:686-696`).
- `fd6659dd5` 2026-06-30 (#6162) — apply tool changes before the next request without dropping run prompt overrides (`packages/agent/src/agent.ts:108-446`).
- `c662ec7e3` 2026-10-01 — restored tools that are not registered yet stay **pending** and activate on registration, until a `setActiveTools()` deactivates them or the next prompt starts.
- Loads are transcript entries (tool changes) so they survive `/tree`, resume, fork (`packages/coding-agent/src/extensions/tool-search/tool.ts:5-9`; `docs/mcp.md:226`).

**Lesson** — Restore dynamic tool loadouts lazily: keep references pending until their providers register.

Related: [[deferred-tool-loading]] · [[mcp-integration]] · [[transcript-carried-system-prompt]] · [[pi--deferred-tool-loading|pi]]
