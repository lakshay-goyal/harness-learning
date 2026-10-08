---
type: tradeoff
concepts: [mcp-integration, code-mode, deferred-tool-loading, replaceable-builtin-extension]
harnesses: [pi, opencode]
---
# mcp-builtin-vs-extension

**Axis**: is MCP part of the core, and are MCP tools declared to the model directly or discovered on demand?

| dimension | pi | opencode | evidence |
|---|---|---|---|
| Stance over time | "pi does not support MCP" (`60e4fcf01` 2025-11-12) → replaceable built-in extension (`8562bcf66` 2026-09-29) | built in since `37c34fd39c` (2025-06-03, "mcp support") | → [[no-builtin-mcp-reversed]] |
| Client | own `packages/mcp` (no official SDK) | official MCP SDK (patched: session recovery, SSE error handling, refresh-token scope) | pi `packages/mcp/README.md:1-5`; opencode `patches/@modelcontextprotocol%2Fsdk@1.29.0.patch`, `c7dee9c609` |
| Default exposure | `codemode`: tools neither declared nor listed; scripts find them with `searchTools()`; alternatives `deferred` / `direct` / `hidden` | declared directly as `<server>_<tool>`; with experimental code mode on, hidden behind `execute` | pi `docs/mcp.md:191-198`; opencode `packages/opencode/src/session/tools.ts:388` → [[code-mode]], [[deferred-tool-loading]] |
| Output cap | 20 KiB middle cut (smaller than built-in 50 KB) | same `Truncate` as built-ins (2000 lines / 50 KB) | [[Constants]] 03-tools |
| Connect timeout | lazy; first prompt waits ≤ 10 s only for direct-tool servers | 30 s connect/list (raised from 5 s, `e5abe1e78b`) | → [[mcp-connect-timeout-too-short]] |
| Project config gating | `.pi/mcp.json` only after project trust | ungated; MCP "outside our trust boundary" (`SECURITY.md:32`) | → [[no-project-trust-gate]] |
| Opt-out | `--no-mcp`, `-builtin:mcp`, or replace with own `/mcp` | don't configure servers | — |

**When each wins**
- **Deferred / code-mode exposure (pi)**: many servers with large tool catalogs; the declaration cost ("13,000-token MCP server description", pre-`60e4fcf01` README) is paid only when a script searches. Tool block stays cache-stable as servers connect.
- **Direct declaration (opencode)**: few servers, models that call tools better than they write orchestration scripts, simpler mental model. Costs: tool block changes when servers connect or `list_changed` fires, and the catalog grows the prompt.

Related: [[mcp-integration]] · [[web-tools-vs-none]] · [[prompt-cache-strategy]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
