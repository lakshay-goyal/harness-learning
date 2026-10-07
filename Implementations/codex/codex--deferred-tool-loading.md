---
type: implementation
harness: codex
concept: deferred-tool-loading
commit: 622e9e3696
files: [codex-rs/tools/src/tool_spec.rs:27-32, codex-rs/tools/src/tool_discovery.rs:6-9, codex-rs/tools/src/tool_executor.rs:51-78, codex-rs/core/src/tools/router.rs:262-281, codex-rs/core/src/tools/handlers/tool_search.rs:197-370, codex-rs/core/src/tools/handlers/tool_search_spec.rs:8-104, codex-rs/core/src/tools/spec_plan.rs:257-292, codex-rs/core/src/tools/spec_plan.rs:392-427, codex-rs/features/src/lib.rs:1518-1529, codex-rs/code-mode-protocol/src/description.rs:16-20]
---
[[deferred-tool-loading]] in [[codex]] — built on the Responses API's native client-executed `tool_search`: the loaded tools are carried **in the transcript item itself**, so there is no harness-side "loaded set".

## Mechanism
- Request declares `{type: "tool_search", execution: "client", description, parameters}` (`codex-rs/tools/src/tool_spec.rs:27-32`). Model emits `tool_search_call` (execution "client"); router parses `SearchToolCallParams` ("failed to parse tool_search arguments: …" on error) (`codex-rs/core/src/tools/router.rs:262-281`).
- Handler runs **BM25** over deferred tool metadata (name, description, namespace, schema-derived search info via `ToolExecutor::search_info`) and returns a `tool_search_output` whose `tools` are loadable specs (`LoadableToolSpec` Function | Namespace, namespaces coalesced) (`codex-rs/core/src/tools/handlers/tool_search.rs:210-370`). The output item persists in history → resume/fork keep the loads (contrast [[deferred-tools-lost-on-resume]] in pi).
- Params: `query` "Search query for deferred tools."; `limit` "Maximum number of tools to return. Defaults to 8." — `TOOL_SEARCH_DEFAULT_LIMIT = 8` (`codex-rs/tools/src/tool_discovery.rs:7`); `limit == 0` → "limit must be greater than zero" (`tool_search.rs:238-243`). Parallel-safe (`tool_search.rs:197`).
- Description (`codex-rs/core/src/tools/handlers/tool_search_spec.rs:36-104`): "# Tool discovery … Searches over deferred tool metadata with BM25 and exposes matching tools for the next model call." + optional source list ("You have access to tools from the following sources: - name: description", bounded by `MAX_TOOL_SEARCH_SOURCE_DESCRIPTION_BYTES = 512 KiB`, `:8`; `ToolSearchSourceListing::Omit` variant drops it) + "For MCP tool discovery, always use `tool_search` instead of `list_mcp_resources` or `list_mcp_resource_templates`." → [[tool-description-design]].
- Exposure levels `ToolExposure::{Direct, Deferred, DeferredModelOnly, DirectModelOnly, CodeModeOnly, Hidden}` (`codex-rs/tools/src/tool_executor.rs:51-78`).
- **Gate is the model, not the user**: if `model_info.supports_search_tool` and a tool is deferrable, it becomes Deferred (direct exposure removed); otherwise deferred exposure is dropped and the tool goes Direct (`codex-rs/core/src/tools/spec_plan.rs:267-276`). `Feature::ToolSearch` and `ToolSearchAlwaysDeferMcpTools` are `Stage::Removed` (`codex-rs/features/src/lib.rs:1518-1529`). MCP tools default to Deferred when search is supported (`codex-rs/core/src/mcp_tool_exposure.rs` `append_mcp_tools`).
- `tool_search` registered only if ≥1 deferred searchable tool exists; any other tool/namespace named `tool_search` is removed and recorded as a collision (`spec_plan.rs:392-427`).
- Failure path returns an **empty** `ToolSearchOutput` (status completed, no tools), not an error (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:338-344`) → [[tool-error-as-result]].
- Visibility decided once at `ToolRouter` creation for every request path (main turn, compaction, side requests) (`85c1500569`).
- Code mode: deferred tools omitted from the `exec` description but present on `tools` / `ALL_TOOLS`; discovery via `await tools.tool_search({query, limit: 8})` returning names + full TypeScript declarations (`codex-rs/code-mode-protocol/src/description.rs:16-20`; `4d15794336`); strict Code-Mode-Only keeps third-party tools deferred (`58ae3ba611`) → [[code-mode]].
- Plugin discovery companion: `list_available_plugins_to_install` / `request_plugin_install` (`tool_discovery.rs:8-9`) for tools not installed at all ([[tool-name-semantics-misfire]]).
- Responses Lite incremental catalogs: later tool additions/removals appended as history items with explicit merge semantics ([[transcript-carried-system-prompt]], [[tool-loadout-stale-within-run]]).

## Constants
| name | value | path:line |
|---|---|---|
| `TOOL_SEARCH_DEFAULT_LIMIT` | 8 | `codex-rs/tools/src/tool_discovery.rs:7` |
| `MAX_TOOL_SEARCH_SOURCE_DESCRIPTION_BYTES` | 512 KiB | `codex-rs/core/src/tools/handlers/tool_search_spec.rs:8` |
| earlier deferral threshold (removed design) | defer MCP tools only when > 100 deferrable tools | `d0047de7cb` commit body |

## Evolution
- 2026-02-09 `becc3a0424` `search_tool` (function tool `search_tool_bm25`); 2026-02-13 `38c442ca7f` limited to Apps, guidance moved into the description.
- 2026-03-11 `77b0c75267` "search_tool migrate to bring you own tool of Responses API"; `ba5b94287e` `tool_suggest`.
- 2026-03-12 `bc48b9289a` connector vocabulary, suppress `list_mcp_resources` for discovery.
- 2026-04-09 `d7f99b0fa6` custom MCPs searchable; 2026-04-14 `78835d7e63` default result caps; 2026-04-17 `d0047de7cb` "always defers all mcp tools rather than deferring once > 100 deferrable tools" (flag, later Removed); 2026-04-18 `6991be7ead` dynamic tools searchable.
- 2026-04-27 `85c1500569` filter dynamic deferred tools from `model_visible_specs` (compaction requests carried hidden tools).
- 2026-08-05 `431c78eb9c` deferred custom (freeform) tools; `fcc4ca552f` longer MCP source descriptions.
- 2026-10-03 `58ca099b03` stable code-mode discovery guidance; `58ae3ba611` strict 3P deferral. 2026-10-06 `4d15794336` ranked discovery inside JS code mode; `e79c498b5e` tool declaration mode preserved across resumed context windows.

## Versus pi
pi: five exposure tiers, own `tool_search` (BM25, limit 8) that activates tools for the next call and records loads as transcript entries ([[pi--deferred-tool-loading]]). codex delegates the load record to the provider protocol item and gates deferral on model capability (`supports_search_tool`) rather than config.
