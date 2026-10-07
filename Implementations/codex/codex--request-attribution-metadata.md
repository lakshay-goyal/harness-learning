---
type: implementation
harness: codex
concept: request-attribution-metadata
commit: 622e9e3696
files: [codex-rs/core/src/responses_metadata.rs:30-61, codex-rs/core/src/responses_metadata.rs:325-360, codex-rs/core/src/responses_metadata.rs:377-405, codex-rs/codex-api/src/requests/headers.rs:5-31, codex-rs/core/src/client.rs:175, codex-rs/core/src/client.rs:945-957, codex-rs/core/src/client.rs:989-993, codex-rs/core/src/client.rs:2336-2353, codex-rs/codex-api/src/common.rs:26-27, codex-rs/login/src/auth/default_client.rs:42, codex-rs/core/src/client_tool_metadata.rs:10-32, codex-rs/response-debug-context/src/lib.rs:5-60]
---
[[request-attribution-metadata]] in [[codex]].

## Mechanism
- **Body `client_metadata`** is a string map on every Responses request (`codex-rs/core/src/responses_metadata.rs:30-61,325-360`). Keys:
  - ids: `installation_id`, `session_id`, `thread_id`, `agent_name`, `turn_id`;
  - context window: `window_id`, `window_number`, `context_window_id`;
  - request kind: `request_kind`, `compaction`;
  - lineage: fork lineage, parent and root turn ids, `subagent_kind`, `thread_source`, `turn_trigger`;
  - policy: `sandbox`/`sandbox_mode`, auto-review flags, `analytics_enabled`, `history_ingest_requested`;
  - `mcp_attribution`, capped at `MAX_MCP_ATTRIBUTION_BYTES` = 16 KiB (`codex-rs/core/src/responses_metadata.rs:46-47`).
  - `tool_namespaces_info`: per-turn tool inventory `namespace → function → {direct, code_mode_name, deferred, source: harness | mcp{server_name}}`; hidden tools skipped; native tool search reported as `tool_search/tool_search_tool` (`codex-rs/core/src/responses_metadata.rs:42`, `:197-222`; `codex-rs/core/src/tools/tool_namespaces_info.rs:16-80`). Added `2b1811e562` 2026-08-07 (#37492), same day the older `code_mode_tool_names` inventory was removed and its key reserved "so callers cannot reintroduce oversized metadata" (`8e4b10446e`; `codex-rs/core/src/responses_metadata.rs:40-41`).
- **User extras** `responses_api_metadata` (config map, `codex-rs/config/src/config_toml.rs:417-418`; `9e301c8c9a` 2026-08-10): ≤ 16 entries, short ASCII keys, values ≤ 128 bytes, reserved harness keys rejected at config load and filtered again at send (`validate_extra_metadata` / `filter_extra_metadata`, `codex-rs/core/src/responses_metadata.rs:496-525`).
- **HTTP compatibility headers** mirror the metadata (`codex-rs/core/src/responses_metadata.rs:377-405`): `x-codex-turn-metadata`, `x-codex-installation-id`, `x-codex-window-id`, `x-codex-parent-thread-id`, and others.
- **Over WebSockets** the same data rides in each request's `client_metadata` (`429cc4860e` 2026-02-19), including the W3C `traceparent`/`tracestate` keys (`codex-rs/codex-api/src/common.rs:26-27`).
- **Fixed headers:**
  - `session-id` and `thread-id` (`build_session_headers`, `codex-rs/codex-api/src/requests/headers.rs:5-14`);
  - `x-openai-subagent: review|compact|memory_consolidation|collab_spawn|<label>` (`codex-rs/codex-api/src/requests/headers.rs:16-31`);
  - `x-codex-beta-features` and `x-codex-turn-state` (`codex-rs/core/src/client.rs:2336-2353`);
  - `OpenAI-Beta: responses_websockets=2026-02-06` on WebSockets (`codex-rs/core/src/client.rs:175`);
  - `originator`, default `codex_cli_rs` (`DEFAULT_ORIGINATOR`, `codex-rs/login/src/auth/default_client.rs:42`), plus a shared User-Agent;
  - `x-oai-attestation` when the provider supports attestation (`codex-rs/core/src/client.rs:126,688`).
- **Kept out of model-visible input and stripped for third parties.**
  - For non-OpenAI providers, internal chat-message metadata passthrough is removed (`codex-rs/core/src/client.rs:942-953`).
  - With `include_internal_metadata = false`, tool-result metadata is cleared (`codex-rs/core/src/client.rs:989-993`).
- **Wire-size guard.** If the serialized request exceeds `MAX_RESPONSE_MESSAGE_BYTES` = 15 MiB, optional executed-tool-call metadata is shed from the wire copy only (`codex-rs/core/src/client_tool_metadata.rs:10-32`).
- **Response side.** `x-request-id`, `x-oai-request-id`, `cf-ray`, `x-openai-authorization-error` and `x-error-json` are captured into `ResponseDebugContext` for errors and bug reports (`codex-rs/response-debug-context/src/lib.rs:5-60`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_MCP_ATTRIBUTION_BYTES` | 16 KiB | `codex-rs/core/src/responses_metadata.rs:47` |
| `MAX_RESPONSE_MESSAGE_BYTES` (soft request cap) | 15 MiB | `codex-rs/core/src/client_tool_metadata.rs:10` |
| `DEFAULT_ORIGINATOR` | `codex_cli_rs` | `codex-rs/login/src/auth/default_client.rs:42` |

## Evolution
- `429cc4860e` 2026-02-19: metadata carried in WebSocket `client_metadata`.
- `bee042d119` 2026-09-16: managed-residency header `x-openai-internal-codex-residency: us`.

## Versus pi
- pi sends attribution headers only when telemetry is enabled ([[pi--http-transport-hardening|pi]]: `provider-attribution.ts`), and impersonates `originator` for Codex OAuth ([[provider-identity-shim]]).
- Codex *is* the originator and sends a rich lineage map on every request. The vendor uses it for routing (`x-openai-subagent`), billing and debugging.

## Failures
- (none recorded)
