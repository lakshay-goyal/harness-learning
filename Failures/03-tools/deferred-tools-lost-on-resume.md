---
type: failure
concepts: [deferred-tool-loading, mcp-integration, auto-compaction]
harnesses: [pi, codex]
---
**Symptom** — MCP tools that `tool_search` had loaded **vanished after resume or `/reload`**; earlier, tools registered dynamically at runtime stayed invisible until `/reload`, and tool refreshes before the next request dropped prompt overrides.

**Root cause** — The tool loadout is persisted state with asynchronous dependencies: the session restored its active set before MCP servers reconnected, so not-yet-registered tools were dropped.

**Fix · [[pi]]**
- `bc2fa8d6d` 2026-03-02 (#1720) — dynamic tool registration refresh (`packages/coding-agent/src/core/agent-session.ts:686-696`).
- `fd6659dd5` 2026-06-30 (#6162) — apply tool changes before the next request without dropping run prompt overrides (`packages/agent/src/agent.ts:108-446`).
- `c662ec7e3` 2026-10-01 — restored tools that are not registered yet stay **pending** and activate on registration, until a `setActiveTools()` deactivates them or the next prompt starts.
- Loads are transcript entries (tool changes) so they survive `/tree`, resume, fork (`packages/coding-agent/src/extensions/tool-search/tool.ts:5-9`; `docs/mcp.md:226`).

**Fix · [[codex]]** (variant: deferred tools leak into side requests)
- Symptom: dynamic deferred tools were filtered only at normal-turn prompt build, so `ToolRouter.model_visible_specs` still included them and compaction requests carried tools that should be hidden (issue #19486).
- `85c1500569` 2026-04-27 — filter at `ToolRouter` creation for every request path (main turn, compaction, side requests).
- `e79c498b5e` 2026-10-06 "Preserve tool declaration mode across resumed context windows"; `58ca099b03` 2026-10-03 "Keep Code Mode tool discovery guidance stable across catalog changes".
- Structural sidestep of the pi symptom: loads are the Responses `tool_search_output` item itself, so resume/fork replay them without a harness-side loaded set (`codex-rs/core/src/tools/handlers/tool_search.rs:210-370`).

**Lesson** — Tool visibility is persisted, request-path-wide state: decide it once upstream of every request builder (turn, compaction, side requests) and restore it lazily on resume, keeping references pending until their providers register.

Related: [[deferred-tool-loading]] · [[mcp-integration]] · [[transcript-carried-system-prompt]] · [[pi--deferred-tool-loading|pi]] · [[codex--deferred-tool-loading|codex]] · [[auto-compaction]]
