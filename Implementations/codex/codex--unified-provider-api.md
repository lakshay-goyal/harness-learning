---
type: implementation
harness: codex
concept: unified-provider-api
commit: 622e9e3696
files: [codex-rs/model-provider-info/src/lib.rs:100-133, codex-rs/codex-api/src/common.rs:279-304, codex-rs/codex-api/src/sse/responses.rs:353-491, codex-rs/codex-api/src/sse/responses.rs:536-585, codex-rs/core/src/client.rs:905-1012]
---
[[unified-provider-api]] in [[codex]].

## Mechanism
- **One wire protocol, no neutral layer.** `WireApi` has exactly one variant, `Responses` (`codex-rs/model-provider-info/src/lib.rs:104-111`). `wire_api = "chat"` fails to deserialize with `CHAT_WIRE_API_REMOVED_ERROR`, which points at discussion #7782 (`codex-rs/model-provider-info/src/lib.rs:100,122-133`). The `ollama-chat` provider id is rejected the same way (`codex-rs/model-provider-info/src/lib.rs:101-102`).
- **The vendor item type is the transcript type.** OpenAI Responses `ResponseItem` values are stored in history, replayed, and sent as `input`. There is no provider-neutral message model and no per-provider converter. Every third-party or local provider must expose a Responses-compatible `/v1/responses` endpoint (Ollama ≥ 0.13.4, LM Studio, Bedrock Mantle; see [[codex--custom-provider-registration|codex custom providers]]).
- **Request struct** `ResponsesApiRequest` (`codex-rs/codex-api/src/common.rs:279-304`): `model, stream, service_tier, input, tools, tool_choice, parallel_tool_calls, reasoning, store, stream_options, include, prompt_cache_key, text, client_metadata, access_programs`.
  - Routing fields (`model`, `stream`, `service_tier`) are declared first, "so gateways may inspect request bodies incrementally before potentially multi-megabyte input" (`codex-rs/codex-api/src/common.rs:281-282`).
  - Absent fields: `max_output_tokens`, `temperature`, `background`, `previous_response_id` (the last exists only on the WebSocket `response.create` variant).
- **Event stream** `ResponseEvent`. The SSE parser maps a whitelist of Responses events and logs the rest at trace level as "unhandled responses event": `function_call_arguments.delta/done`, `content_part.*`, `in_progress`, `output_text.done` and others (`codex-rs/codex-api/src/sse/responses.rs:480-491`).
  - Tool calls come only from the completed item in `response.output_item.done` (`codex-rs/codex-api/src/sse/responses.rs:353-358`). Argument deltas are never parsed (see [[streaming-json-repair]]).
  - `custom_tool_call_input.delta` is forwarded as `ToolCallInputDelta` for live UI (`codex-rs/codex-api/src/sse/responses.rs:366-375`).
  - An event that cannot be parsed is logged and skipped, not treated as fatal (`codex-rs/codex-api/src/sse/responses.rs:573-585`).
  - A stream that ends without `response.completed` is an error ([[terminal-event-required]]; `codex-rs/codex-api/src/sse/responses.rs:557-563`).
- **Provider runtime abstraction** (`a803790a10` 2026-04-16): there is a provider trait for auth, capabilities, the models endpoint and preferred side models, but every provider still speaks Responses.
- **Non-OpenAI sanitization** is minimal. It strips internal chat-message metadata passthrough and clears `encrypted_function_args` on FunctionCall items (`codex-rs/core/src/client.rs:942-953`), and tool-result metadata is removed when `include_internal_metadata` is false (`codex-rs/core/src/client.rs:989-993`).

## Evolution
- 2025-04: Responses-first (TypeScript CLI era).
- `e924070cee` 2025-05-08 (#862): Rust CLI adds Chat Completions. `b940adae8e` the same day fixes Responses (#872).
- Chat-path fixes:
  - `d7245cbbc9` 2025-06-02: tools passed.
  - `acb8ed493f` 2025-12-08: a missing tool name crashed LiteLLM.
  - `649badd102` and `5f80ad6da8`: multiple and parallel tool calls on chat.
  - `de1768d3ba` 2025-11-17: Claude empty `finish_reason`.
  - `bba5e5e0d4` 2026-01-05: the `[DONE]` sentinel.
  - All of these are in [[stop-reason-mapping-gaps]].
- `43e6e75317` 2025-12-11 (#7897): deprecation notice, "will be removed in early Feb 2026", shown once per invocation.
- `fe03320791` 2026-01-13 (#8798): Ollama built-ins switched to Responses.
- `d75626ad99` 2026-01-12: Responses over WebSocket (see [[codex--session-affinity-cache-routing|codex WS]]).
- `d2394a2494` 2026-02-03 "chore: nuke chat/completions API (#10157)": deleted `9257d8451c:codex-rs/codex-api/src/requests/chat.rs` (494 lines) and `9257d8451c:codex-rs/codex-api/src/sse/chat.rs` (717 lines). `88598b9402` 2026-02-03 (#10498) drops `wire_api` from clients.
- `a803790a10` 2026-04-16: provider runtime abstraction. `cefcfe43b9` 2026-04-20: Amazon Bedrock built-in, the first non-OpenAI hosted provider (Responses-compatible Mantle endpoint).

## Quirks
- A `response.incomplete` with a reason other than `interrupted` or `content_filter` (e.g. `max_output_tokens`) becomes a *retryable* `Stream` error "Incomplete response returned, reason: …" (`codex-rs/codex-api/src/sse/responses.rs:418-433`). A length stop is retried, not continued.
- There is no stop-reason enum to map, because the Responses item/status model is consumed directly.

## Versus pi
- [[pi--unified-provider-api|pi]] normalizes 10 chat APIs and 42 providers into its own event protocol. Codex converges on one vendor protocol and deleted its second protocol rather than maintain quirk tables. See [[provider-breadth]] and [[no-chat-completions-wire]].

## Failures
- [[stop-reason-mapping-gaps]]
- [[endpoint-rejects-request-field]]
