---
type: concept
stage: tool-design
tier: candidate
aliases: [tool_search, "exposure: direct/model-only/codemode/deferred/hidden", ToolExposure, Bm25Ranker, prepareLoadout, deferred-tool-discovery, deferred-tool-exposure, tool-exposure-levels, bm25-tool-search, tool-catalog-budget, tool_search_output, defer_loading, supports_search_tool]
harnesses: [pi, codex]
---
Register tools without declaring them to the model; give each a reachability tier, and let the model discover and load the ones it needs via a search tool, with loaded tools declared from the next call on.

## Why
- Large tool catalogs (MCP servers) cost thousands of tokens per request and dilute selection.
- Declaring tools that appear asynchronously (servers connecting) churns the prompt prefix and cache ([[mcp-startup-blocks-and-description-churn]], [[late-tool-change-rewrites-cache]]).
- Loaded tool sets are state: they must survive resume/branching and restore even when the tool registers late ([[deferred-tools-lost-on-resume]]).

## Design space
- Declare everything (baseline).
- **Exposure tiers**: declared+callable / declared-only (orchestrators) / script-only / searchable / hidden (pi's five).
- Discovery: lexical BM25 over name/description/schema/namespace (pi) vs embeddings/hybrid (pi leaves a `ToolRanker` seam) vs provider-native tool search.
- Activation timing: next call (pi) vs same call.
- Persistence of loads: transcript entries replayed on resume/fork (pi, since 2026-09) vs message-anchored loading (pi's earlier `addedToolNames`, removed).
- Catalog budget: list cheapest tools round-robin per namespace until a token budget (pi codemode, 3000 tok) vs never list.
- Stable discovery-tool description that never enumerates the searchable set (pi).
- Alternative: code-mode scripts call undeclared tools directly → [[code-mode]].
- **Provider-native client-executed tool search; the loaded tools live in the protocol output item in history** (✔ codex `{type:"tool_search", execution:"client"}`) — no harness-side loaded set to restore.
- Deferral gated by model capability, not user config (✔ codex `supports_search_tool`; flags `ToolSearch`/`ToolSearchAlwaysDeferMcpTools` Removed); earlier design deferred MCP tools only when > 100 deferrable tools (`d0047de7cb`).
- Six exposure levels incl. `DeferredModelOnly` and `CodeModeOnly` (✔ codex) vs five (✔ pi).
- Failed search → empty result, not an error (✔ codex).
- Visibility decided once at router creation for every request path incl. compaction/side requests (✔ codex `85c1500569`).
- Separate install-discovery tool for tools not installed at all (✔ codex `request_plugin_install`) → [[tool-name-semantics-misfire]].

## Implementations
- [[pi--deferred-tool-loading|pi]] — `ToolExposure` 5 tiers, `tool_search(query, limit=8)` BM25 → `setActiveTools` for the next call, loads recorded in transcript; MCP default exposure codemode.
- [[codex--deferred-tool-loading|codex]] — Responses `tool_search` (client-executed, BM25, limit 8); MCP/dynamic tools deferred when the model supports search; loads persist as `tool_search_output` items.

## Failures
- [[deferred-tools-lost-on-resume]]
- [[mcp-startup-blocks-and-description-churn]]
- related (06): [[late-tool-change-rewrites-cache]]
- [[tool-name-semantics-misfire]]

## Related
[[code-mode]] · [[mcp-integration]] · [[minimal-default-toolset]] · [[cache-stable-prompt-prefix]] · [[transcript-carried-system-prompt]] · [[tool-description-design]] · [[plugin-tools]] · [[tool-wire-kinds]]
