---
type: implementation
harness: codex
concept: session-affinity-cache-routing
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:286-308, codex-rs/core/src/client.rs:337-414, codex-rs/core/src/client.rs:581-603, codex-rs/core/src/client.rs:961, codex-rs/core/src/client.rs:1003, codex-rs/core/src/client.rs:1401-1434, codex-rs/core/src/client.rs:1563-1625, codex-rs/core/src/client.rs:1976-1994, codex-rs/core/src/client.rs:2336-2353, codex-rs/codex-api/src/sse/responses.rs:210-218, codex-rs/codex-api/src/endpoint/responses_websocket.rs:166-168, codex-rs/codex-api/src/endpoint/responses_websocket.rs:632-647, codex-rs/core/src/compact.rs:282-284]
---
[[session-affinity-cache-routing]] in [[codex]].

## Mechanism
- **Stateless on the API.** Every request is `store: false` and resends the full `input` with `include: ["reasoning.encrypted_content"]` (`codex-rs/core/src/client.rs:961,1003`). This was a design decision in `591cb6149a` 2025-07-23, "Always send entire request context (#1641)": "Request encrypted COT when not storing Responses. Send entire input context instead of sending previous_response_id."
- **Cache key** `prompt_cache_key` is always sent. Resolution order (`codex-rs/core/src/client.rs:581-593`):
  1. explicit `prompt_cache_key_override`;
  2. for `SessionSource::Internal` sessions with a parent thread → `"{source}:{parent_thread_id}"`;
  3. otherwise the **session id**, shared by a root agent and its subagents.
- **Affinity header.** "ChatGPT derives cache affinity from the Responses session-id header" (`codex-rs/core/src/client.rs:595-596`).
  - For root agents the `session-id` header carries the prompt cache key.
  - For non-root agents it carries the real session id (`codex-rs/core/src/client.rs:597-603`).
  - Ephemeral forks inherit the parent session id as their cache key, so they hit the parent's cache (`bc5957eac9`).
- **Sticky routing within a turn** (`x-codex-turn-state`):
  - The server returns the token on the first response of a turn: an SSE header, the WebSocket handshake, or a `response.metadata` event (`codex-rs/codex-api/src/sse/responses.rs:210-218`).
  - The client stores it in a per-turn `OnceLock` (`turn_state`, `codex-rs/core/src/client.rs:308`) and echoes it on every request of the same turn: retries, tool follow-ups, compaction (header `codex-rs/core/src/client.rs:2336-2353`; WebSocket `client_metadata` `codex-rs/core/src/client.rs:1976-1977`).
  - It is never carried across turns; a new turn resets it (`codex-rs/core/src/client.rs:1594`).
  - `ModelClientSession` is turn-scoped and reused across retries (`codex-rs/core/src/session/turn.rs:411-412`). Compaction reuses one session, so sticky routing and WebSocket state survive its retries (`codex-rs/core/src/compact.rs:282-284`).
- **WebSocket incremental mode** is the only use of `previous_response_id`:
  - The per-turn session keeps the last request and the items the last response added.
  - The next request becomes `previous_response_id` plus the delta items only if both of these hold:
    - every non-input property is identical. `responses_request_properties_match` destructures `ResponsesApiRequest` exhaustively, so the compiler forces a decision for any new field (`codex-rs/core/src/client.rs:341-393`);
    - the new input strictly extends old input plus old response output, compared item by item ignoring internal metadata (`get_incremental_items`, `codex-rs/core/src/client.rs:395-414,1401-1434`). A late change to tool-result metadata forces a full resend.
  - Otherwise a full `response.create` is sent with a recorded reset reason `incremental|other|restored_history|no_previous_request` (`codex-rs/core/src/client.rs:1979-1994`).
- **Server state is per connection.**
  - `previous_response_not_found` maps to retryable "Previous response was not found. Retrying the full request." (`codex-rs/codex-api/src/endpoint/responses_websocket.rs:166-168,632-647`).
  - The connection and its continuation reset when the auth owner, the provider/auth revision key, or the socket's closed state changes (`codex-rs/core/src/client.rs:1563-1625`).
- **Prewarm.** A `generate=false` `response.create` warms the connection and server-side response state before the first request. See [[codex--cache-warming|codex cache warming]].
- **Prefix bytes** are kept identical by deterministic ids (UUIDv5) for base instructions and tool items ([[codex--cache-stable-prompt-prefix|codex prefix]]). Changed instructions force a full request without `previous_response_id` (`c9253c4977`).

## Constants
| name | value | path:line |
|---|---|---|
| WebSocket beta header | `responses_websockets=2026-02-06` | `codex-rs/core/src/client.rs:175` |
| cache key default | session id (`"{source}:{parent_thread_id}"` for internal sessions) | `codex-rs/core/src/client.rs:581-593` |
| `PREVIOUS_RESPONSE_NOT_FOUND_CODE` | `previous_response_not_found` | `codex-rs/codex-api/src/endpoint/responses_websocket.rs:166` |

## Evolution
- `591cb6149a` 2025-07-23: always send full context and request encrypted CoT. `previous_response_id` was dropped for HTTP.
- `6a6bf99e2c` 2025-08-11 (#2200): send `prompt_cache_key`.
- `d75626ad99` 2026-01-12: reuse the WebSocket connection. `e726a82c8a` 2026-01-12: append/incremental requests.
- `ebdd8795e9` 2026-01-16 (#9332): `x-codex-turn-state` sticky routing.
- `3b1cddf001` 2026-01-29: HTTP fallback.
- `e416e578bb` 2026-02-06: preconnect. `1fbf5ed06f` 2026-02-06: v2 beta header.
- `0639c33892` 2026-02-10: compare the full request ("Tools can dynamically change mid-turn now").
- `4473147985` 2026-02-10: don't resend output items already in server context.
- `429cc4860e` 2026-02-19: metadata carried in WebSocket `client_metadata`.
- `69df12efb3` 2026-03-03: WebSocket v1 removed.
- `770616414a` 2026-03-17: prefer WebSockets when supported.
- `2070d5bfd3` 2026-05-06: `response.processed` ack; removed `d312a53e2a` 2026-06-04.
- `17d552fb4d` 2026-05-18: compaction no longer resets WebSocket state externally, since the strict-extension check already covers it.
- `4ce563a873` / `3307240195` 2026-05-28: Guardian reviewer sessions get their own stable key.
- `f297b9f07d` 2026-06-13: turn-state sent through compact requests.
- `4aa950d456` 2026-07-14 (#33035): cache key thread id → session id, so root and subagent requests with API-key auth share one key.
- `64dc1c7a01` 2026-07-22 (#34763): retry with the full request when the previous response is missing.
- `bc5957eac9` 2026-09-11 (#44862): ephemeral forks keep the parent's cache affinity.
- `3f4668da20` 2026-09-27 (#48812): history-aware prewarming for idle threads.
- `c9253c4977` 2026-10-05 (#51156): base instructions as input messages with stable ids.

## Quirks
- **Two identities.** The body `prompt_cache_key` and the `session-id` header can differ (non-root agents), because the ChatGPT backend routes by header, not by body key.
- **Opposite fork policy to pi.** pi mints a fresh key for forks and side requests ([[side-request-cache-pollution]]). Codex deliberately *shares* the parent's key with subagents, ephemeral forks and internal sessions, because they share the prefix.

## Versus pi
- [[pi--session-affinity-cache-routing|pi]] implements the same Codex WebSocket protocol from the client side:
  - delta + `previous_response_id`;
  - `previous_response_not_found` → one full retry;
  - sticky SSE fallback;
  - a 55 min socket rotation.

  Codex is the reference implementation. It adds an exhaustive property-equality check that fails closed on new fields, a per-turn sticky routing token, `generate=false` warmup, and cache-key sharing across the agent family.
- See [[cache-strategy]].

## Failures
- [[cache-key-scoped-to-wrong-identity]]
- [[incremental-request-diverges-from-history]]
- [[persistent-connection-lifetime-exceeded]]
- [[server-side-state-missing-on-continuation]]
- [[transport-fallback-after-partial-output]]
