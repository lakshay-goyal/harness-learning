---
type: implementation
harness: codex
concept: mcp-integration
commit: 622e9e3696
files: [codex-rs/codex-mcp/src/rmcp_client.rs:95-107, codex-rs/codex-mcp/src/connection_manager.rs:358-367, codex-rs/codex-mcp/src/mcp/mod.rs:196-197, codex-rs/codex-mcp/src/mcp/mod.rs:575-593, codex-rs/codex-mcp/src/connection_manager/tool_catalog.rs:286-345, codex-rs/codex-mcp/src/tool_catalog_cache.rs:33-34, codex-rs/codex-mcp/src/pagination.rs:9-13, codex-rs/codex-mcp/src/tools.rs:22, codex-rs/codex-mcp/src/tools.rs:120-316, codex-rs/codex-mcp/src/elicitation.rs:1-7, codex-rs/codex-mcp/src/elicitation.rs:480-560, codex-rs/core/src/mcp_tool_call.rs:125, codex-rs/core/src/mcp_tool_call.rs:853-1001, codex-rs/core/src/mcp_tool_call.rs:1245-1255, codex-rs/core/src/mcp_tool_call.rs:1457-1461, codex-rs/core/src/mcp_tool_call.rs:2461-2492, codex-rs/core/src/mcp_tool_exposure.rs:18-19, codex-rs/core/src/tools/handlers/mcp.rs:142-150, codex-rs/protocol/src/models.rs:2304-2338, codex-rs/core/src/tools/spec_plan.rs:199-292]
---
[[mcp-integration]] in [[codex]] — MCP is core from the Rust era (2025-05): own crate stack, non-blocking startup with a 1 s grace, cached catalogs, `mcp__<server>__` namespaces with hashed collision suffixes, deferred exposure via `tool_search`, per-tool approvals from annotations.

## Mechanism
### Crates
- `codex-rs/codex-mcp` (connection set, catalog, naming, elicitation, OAuth/auth changes, Codex Apps connectors), `codex-rs/rmcp-client` (transports: local stdio, executor-process stdio, streamable HTTP with retry, in-process; OAuth incl. enterprise ID-JAG), `codex-rs/ext/mcp` (plugin-contributed MCP providers / stream manager); core glue `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/src/mcp_tool_exposure.rs`, `codex-rs/core/src/tools/handlers/mcp.rs`.
- Codex Apps = connector MCP server `codex_apps` (app metadata/directory cache; per-app `omit_tools_from`) (`codex-rs/core/src/connectors.rs`, `codex-rs/connectors`).

### Startup & catalogs
- Timeouts: `DEFAULT_STARTUP_TIMEOUT = 30 s`, `DEFAULT_TOOL_TIMEOUT = 300 s`; per-server `startup_timeout_sec` / `tool_timeout_sec` (`codex-rs/codex-mcp/src/rmcp_client.rs:106-107`; `connection_manager.rs:358-367`).
- **Optional servers don't block**: shared `DEFAULT_OPTIONAL_MCP_STARTUP_GRACE = 1 s` (configurable), then omitted from that turn's catalog while they keep starting (`codex-rs/codex-mcp/src/mcp/mod.rs:196-197`; `connection_manager/tool_catalog.rs:286-345`). Must-wait: `required = true` servers (`connection_manager/required.rs`), servers needed by a mentioned plugin / skill / `mcp://` mention, `codex_apps` without cache (`tool_catalog.rs:297-306`).
- **Catalog cache**: cached definitions exposed before startup finishes, read-only hint cleared (may be stale), calls wait for the live server (`3bbf1fe757`); capacity 32, TTL 30 min (`codex-rs/codex-mcp/src/tool_catalog_cache.rs:33-34`); capability `codex/tool-catalog-cache` ("Experimental… development and testing only"; `cacheable: false` disables sharing) (`rmcp_client.rs:98-101`). Cached catalogs satisfy required-server readiness (`b62fcd3c4b`).
- Pagination caps: ≤100 pages, ≤2048 items/server (8192 for `codex_apps`), cursor ≤64 KiB, 30 s pagination timeout (`codex-rs/codex-mcp/src/pagination.rs:9-13`).
- Interrupt during startup: tool list + router built per step snapshot under the turn cancellation token; user input recorded first (`d7e8f4c3dc`) → [[side-phase-input-lost]]. Stdio servers run in their own process group and are cleaned up (`82c981cafc`, "orphan process storms") → [[process-tree-kill]].

### Naming
- Model name = namespace `mcp__<server>__` + tool (`LEGACY_MCP_TOOL_NAME_PREFIX`, `codex-rs/codex-mcp/src/tools.rs:22,225-233`); chars outside `[A-Za-z0-9_]` → `_` (`codex-rs/codex-mcp/src/mcp/mod.rs:575-593`); collisions after sanitization get a 12-hex SHA-1 suffix of the raw identity (`server\0namespace\0connector\0callable\0raw`); total ≤ 128 bytes with truncate+hash (`tools.rs:120-316`). Per-server opt-out of the prefix (`74e9d7efc4`, `Feature::NonPrefixedMcpToolNames`). Duplicate effective names can be a hard pre-sampling error (`error_on_tool_collisions`, `1e489adad0`) → [[mcp-tool-name-collision]].

### Exposure & schemas
- `apply_mcp_tool_exposure_policy` (`codex-rs/core/src/tools/spec_plan.rs:199-292`): `omit_tools_from` per server/app removes direct/deferred/code-mode exposure; when the model supports `tool_search`, MCP tools default to **Deferred** (`codex-rs/core/src/mcp_tool_exposure.rs` `append_mcp_tools`) → [[deferred-tool-loading]]; code-mode namespaces excludable → [[code-mode]].
- Schema budget per server `tool_input_schema_max_bytes`; agent-plugin MCP: 8000 B/spec (else open schema), 64000 B total (`mcp_tool_exposure.rs:18-19`) → [[tool-schema-normalization]].
- Resources: three generic tools `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource`, present whenever any MCP server is configured (`codex-rs/core/src/tools/handlers/mcp_resource/*.rs`); parallel-safe; the `tool_search` description tells the model not to use them for tool discovery.

### Calls & results
- Request `_meta` enrichment: sandbox state (server capability `codex/sandbox-state-meta`), thread / session / window ids, item id (`rmcp_client.rs:97`; `codex-rs/core/src/mcp_tool_call.rs:853-912,1245-1255`).
- Result conversion: `structuredContent` present → serialized as the text output (content blocks dropped); else content blocks → text/image items (`codex-rs/protocol/src/models.rs:2304-2338`); images/audio → "<image content omitted because you do not support image input>" for non-multimodal models (`mcp_tool_call.rs:948-985`) → [[structured-tool-output]]; header `Wall time: X seconds\nOutput:` (`codex-rs/core/src/tools/context.rs:186-200`).
- Size bounds: oversized serialized results → bounded text preview preserving `isError` (`codex-rs/utils/output-truncation/src/lib.rs:43-89`); event copies capped `MCP_TOOL_CALL_EVENT_RESULT_MAX_BYTES = 1 MiB` (`mcp_tool_call.rs:125,987-1001`); rollouts truncated (`3516cb9751`); paginated history preview 64 KiB (`820f85cf59`) → [[unbounded-tool-output-overflows-context]].
- Parallelism: only `readOnlyHint == true` or server opt-in `supports_parallel_tool_calls` (`codex-rs/core/src/tools/handlers/mcp.rs:142-150`) → [[unannotated-mcp-tools-serialized]].

### Approvals & elicitation
- Auto mode: `destructiveHint == true` → approval; else `readOnlyHint == true` → none; else approval if destructive or openWorld, both defaulting to true when missing (`mcp_tool_call.rs:2461-2478`); per-tool modes Prompt / Writes / Approve (`:2480-2492`) → [[tool-safety-annotations]], [[mcp-annotation-defaults-unsafe]].
- Prompt options "Allow", "Allow for this session", "Allow and don't ask me again", "Cancel" (`mcp_tool_call.rs:1457-1461`), keyed per session / persistent (`:1730-1760`); persistent allow = `ApprovedMcpPolicyAmendment` → [[approval-policy-modes]].
- Elicitation (`codex-rs/codex-mcp/src/elicitation.rs:1-7,480-560`): auto-declined when `auto_deny`, for tool-suggestion kinds, without authority/permission profile, or by granular policy; otherwise surfaced as protocol events and resolved later; strict-auto-review elicitations go to the reviewer agent ([[llm-approval-reviewer]]); `Feature::ToolCallMcpElicitation` stable/on (`codex-rs/features/src/lib.rs:1878`). Elicitation time pauses unified-exec timers ([[elicitation-pause]]).
- Tool ordering kept deterministic for prompt caching ([[nondeterministic-tool-order-breaks-cache]]).
- **MCP auth/startup config keys** (`codex-rs/config/src/config_toml.rs:300-324`): `mcp_oauth_credentials_store` = keyring / file / auto (default: keyring if available, else file in `CODEX_HOME`); `mcp_oauth_callback_port` (fixed port for the local OAuth callback, else ephemeral); `mcp_oauth_callback_url` (override redirect URI; listener still binds 127.0.0.1); `mcp_optional_startup_grace_ms` (default 1000 ms shared wait for *optional* servers while building the initial tool catalog; 0 = wait each server's `startup_timeout_sec`); `mcp_enterprise_managed_auth` (`:300`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_STARTUP_TIMEOUT` / `DEFAULT_TOOL_TIMEOUT` | 30 s / 300 s | `codex-rs/codex-mcp/src/rmcp_client.rs:106-107` |
| `DEFAULT_OPTIONAL_MCP_STARTUP_GRACE` | 1 s | `codex-rs/codex-mcp/src/mcp/mod.rs:197` |
| `MAX_TOOL_NAME_LENGTH` / `CALLABLE_NAME_HASH_LEN` | 128 bytes / 12 hex (SHA-1) | `codex-rs/codex-mcp/src/tools.rs:226-227` |
| pagination: pages / items / apps items / cursor / timeout | 100 / 2048 / 8192 / 64 KiB / 30 s | `codex-rs/codex-mcp/src/pagination.rs:9-13` |
| tool catalog cache | 32 entries, 30 min TTL | `codex-rs/codex-mcp/src/tool_catalog_cache.rs:33-34` |
| event result cap | 1 MiB | `codex-rs/core/src/mcp_tool_call.rs:125` |
| agent-plugin MCP spec / total budget | 8000 / 64000 bytes | `codex-rs/core/src/mcp_tool_exposure.rs:18-19` |
| persisted history preview | 64 KiB | `820f85cf59` |

## Evolution
- 2025-05-02 → 05-05 `83961e0299` mcp-types, `21cd953dbd` mcp-server, `2cf7aeeeb6` mcp-client. 2025-09-26 `e555a36c6a` rmcp-client (official Rust SDK); legacy client removed 2025-10-22 `4cd6b01494`.
- 2025-10-19 `0170860ef2` prefix MCP tool names with `mcp__` (origin clear to the model).
- 2026-01-22 `a2c829a808` connectors (apps) via MCP.
- 2026-02-06 `82c981cafc` process-group cleanup for stdio servers. 2026-02-20 `d3cf8bd0fa` destructive tools require approval; 2026-03-25 `32c4993c8a` unannotated → pessimistic defaults.
- 2026-04-03 `a3b3e7a6cc` hyphenated server names. 2026-04-09 `d7f99b0fa6` tool search over custom MCPs. 2026-04-30 `3516cb9751` truncate MCP outputs in rollouts.
- 2026-05-11 `32b1ae7099` built-in MCPs dropped ("Drop something that was never used"); 2026-05-21 `c83ba22359` read-only tools parallel.
- 2026-06-15 `41db093aa0` tool timeout 120 → 300 s. 2026-06-23 `55bc38a22b` Codex Apps auth elicitation hang.
- 2026-07-13 `8b2c84ddcc` startup timeouts applied during client creation. 2026-07-22 `d7e8f4c3dc` input preserved when startup interrupted. 2026-07-23 `74e9d7efc4` per-server prefix opt-out. 2026-07-27 `3bbf1fe757` cached tools before startup. 2026-07-28 `d9e1c9cd55` "Avoid blocking turns on optional MCP startup" ("A pending optional MCP server can delay the first model request even when the turn does not need that server").
- 2026-08-05 `1e489adad0` strict collision errors. 2026-08-20 `1bfabb21fe` name limit 64 → 128 bytes. 2026-08-27 `124e560b93` configurable grace.
- 2026-09-05 `531f3836a1` `codex mcp-server` (Codex as MCP server) removed in favour of app-server. 2026-09-09 `3436cad5ab` elicitation cancellation + reset on reconnect. 2026-09-24 `b62fcd3c4b` cached catalogs satisfy readiness; `339e981ba7` configurable schema budgets. 2026-10-02 `820f85cf59` 64 KiB history preview.

## Versus pi
pi had no MCP for ~10 months ([[no-builtin-mcp-reversed]]), then a replaceable built-in extension: codemode-only default exposure, ≤64-char names with sha256-8 suffix, 60 s call timeout, 20 KB middle-cut output, 10 s first-prompt wait only for direct tools ([[pi--mcp-integration]]). codex: deferred-by-default (if model supports search), 128-byte names, 300 s timeout, 1 s optional grace + catalog cache, approval prompts from annotations, `_meta` sandbox state.

## Failures
[[mcp-startup-blocks-and-description-churn]] · [[mcp-tool-name-collision]] · [[unannotated-mcp-tools-serialized]] · [[unbounded-tool-output-overflows-context]] · [[mcp-annotation-defaults-unsafe]] · [[nondeterministic-tool-order-breaks-cache]] · [[side-phase-input-lost]] · [[bash-descendants-hang-or-lose-output]]
