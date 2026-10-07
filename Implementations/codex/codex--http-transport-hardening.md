---
type: implementation
harness: codex
concept: http-transport-hardening
commit: 622e9e3696
files: [codex-rs/model-provider-info/src/lib.rs:66-75, codex-rs/model-provider-info/src/lib.rs:304-308, codex-rs/model-provider-info/src/lib.rs:520-523, codex-rs/codex-api/src/sse/responses.rs:418-433, codex-rs/codex-api/src/sse/responses.rs:536-585, codex-rs/codex-api/src/endpoint/responses_websocket.rs:164-168, codex-rs/codex-api/src/endpoint/responses_websocket.rs:632-647, codex-rs/codex-api/src/endpoint/responses_websocket.rs:693, codex-rs/codex-api/src/endpoint/responses_websocket.rs:897-903, codex-rs/core/src/client.rs:175, codex-rs/core/src/client.rs:1030-1038, codex-rs/core/src/client.rs:1634-1643, codex-rs/core/src/client.rs:2296-2310, codex-rs/core/src/client.rs:2365, codex-rs/websocket-client/src/dialer.rs:34, codex-rs/http-client/src/outbound_proxy.rs:29-30, codex-rs/http-client/src/route_aware_redirect.rs:31]
---
[[http-transport-hardening]] in [[codex]].

## Mechanism
- **Transports:** Responses over SSE (HTTP) or over a WebSocket. The WebSocket uses v2 protocol header `OpenAI-Beta: responses_websockets=2026-02-06` (`codex-rs/core/src/client.rs:175`). WebSockets are preferred when the provider sets `supports_websockets`. Own crates handle the plumbing:
  - `codex-rs/http-client`: custom CA, outbound proxy, route-aware pool, TLS fallback, Retry-After parsing, response size limits.
  - `codex-rs/websocket-client`: a proxy-aware dialer with happy-eyeballs.
- **Stream idle timeout:**
  - Each SSE `next()` is wrapped in `timeout(idle_timeout)`. On expiry the stream fails with `Stream("idle timeout waiting for SSE")`, which is retryable (`codex-rs/codex-api/src/sse/responses.rs:536-569`).
  - The default is `stream_idle_timeout_ms` = 300_000 (`codex-rs/model-provider-info/src/lib.rs:66`).
  - The same bound applies to WebSocket receive and, since `35aaa5d9fc`, to WebSocket *send* (`codex-rs/codex-api/src/endpoint/responses_websocket.rs:693,897-903`).
- **Connect timeout:** `websocket_connect_timeout_ms` defaults to 15_000 (`DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS`; `codex-rs/model-provider-info/src/lib.rs:71,520-523`). It was introduced as `websocket_startup_timeout_ms` so that prewarm cannot block `turn/start`.
- **Server lifetime and lost state are retryable:**
  - The server caps a Responses WebSocket at 60 min and sends `websocket_connection_limit_reached`. Codex maps this to a retryable error, and the next attempt opens a new socket.
  - `previous_response_not_found` maps to retryable "Previous response was not found. Retrying the full request."
  - Code: `codex-rs/codex-api/src/endpoint/responses_websocket.rs:164-168,632-647`.
- **Connection reset** happens when the auth owner, the provider/auth revision key, or the socket's closed state changes (`codex-rs/core/src/client.rs:1563-1625`).
- **Sticky WebSocket → HTTPS fallback:**
  - The `force_http_fallback` / `disable_websockets` atomic is session-wide.
  - It is triggered by stream-retry exhaustion or `426 UPGRADE_REQUIRED` (`codex-rs/core/src/client.rs:1030-1038,2296-2310`).
  - The fallback resets the retry counter but still waits for any server Retry-After deadline. It emits a Warning, "Falling back from WebSockets to HTTPS transport." (`codex-rs/core/src/responses_retry.rs:120-139`).
- **Terminal event required:** "stream closed before response.completed" (`codex-rs/codex-api/src/sse/responses.rs:557-563`).
- **Incomplete responses:**
  - `response.incomplete` with reason `content_filter` → ContentFilter.
  - Any other reason except `interrupted` → a retryable Stream error (`codex-rs/codex-api/src/sse/responses.rs:418-433`).
- **Body compression:** zstd, only when the feature is on, auth is ChatGPT/Codex-backend, and the provider is OpenAI (`codex-rs/core/src/client.rs:1634-1643`).
- **Bedrock:** the AWS provider cannot be combined with WebSockets, and config validation rejects the combination (`codex-rs/model-provider-info/src/lib.rs:304-308`).
- **Retry UI noise.** The first WebSocket retry notification is hidden in release builds (`d391f3e2f9` 2026-02-11; `codex-rs/core/src/responses_retry.rs:145-159`). After that, the stream-error event "Reconnecting... {n}/{max}" is shown.
- **Unbounded network wait** (feature `UnboundedConnectionRetries`): `ConnectionFailed` on user, non-Bedrock sampling requests retries forever. The delay starts at 5 s and doubles to a 60 s cap, the UI shows "Reconnecting... waiting for network", and the normal retry budget is not consumed (`codex-rs/core/src/responses_retry.rs:23-24,93-118`). Details are in [[auto-retry-backoff]].

## Constants
| name | value | path:line |
|---|---|---|
| `stream_idle_timeout_ms` default | 300_000 ms | `codex-rs/model-provider-info/src/lib.rs:66` |
| `websocket_connect_timeout_ms` default | 15_000 ms | `codex-rs/model-provider-info/src/lib.rs:71` |
| server WebSocket lifetime | 60 min (server-enforced, `websocket_connection_limit_reached`) | `codex-rs/codex-api/src/endpoint/responses_websocket.rs:165` |
| `RESPONSE_STREAM_CHANNEL_CAPACITY` | 1600 events | `codex-rs/core/src/client.rs:2365` |
| WebSocket happy-eyeballs delay | 250 ms | `codex-rs/websocket-client/src/dialer.rs:34` |
| system proxy cache TTL | 60 s success / 5 s unavailable | `codex-rs/http-client/src/outbound_proxy.rs:29-30` |
| max redirects | 10 | `codex-rs/http-client/src/route_aware_redirect.rs:31` |
| connection-wait retry delay | 5 s doubling to 60 s | `codex-rs/core/src/responses_retry.rs:23-24` |

## Evolution
- `9846adeabf` 2025-07-18: stream idle timeout and retry config.
- `d75626ad99` 2026-01-12: reuse the WebSocket connection.
- `3b1cddf001` 2026-01-29: HTTP fallback, because "not all proxies work with websockets".
- `e416e578bb` 2026-02-06: preconnect.
- `1fbf5ed06f` 2026-02-06: v2 beta header.
- `a94505a92a` 2026-02-07: permessage-deflate.
- `6d08298f4e` 2026-02-07: fall back on `426 UPGRADE_REQUIRED`.
- `69df12efb3` 2026-03-03: v1 removed, "V2 is the way to go!".
- `6ea041032b` 2026-03-17 (#14838): 15 s WebSocket startup timeout. Prewarm had blocked `turn/start` "for up to five minutes".
- `770616414a` 2026-03-17: prefer WebSockets when the provider supports them.
- `35aaa5d9fc` 2026-05-01 (#20751): bound WebSocket sends with the idle timeout. Sessions had "recover[ed] only after a long quiet period when the server had already logged the websocket as disconnected".
- `2070d5bfd3` 2026-05-06: `response.processed` ack added; removed `d312a53e2a` 2026-06-04.
- `64dc1c7a01` 2026-07-22 (#34763): `previous_response_not_found` becomes retryable with the full request.
- `5a0d0929e2` 2026-08-07: keep streams alive through connection failures. Made configurable in `da898490fc` 2026-08-14.
- `7769bccbb2` 2026-09-07: avoid WebSocket waits in Guardian classification.

## Quirks
- The idle timeout is 5 min, so stall detection is slow; it is tuned to tolerate long reasoning gaps.
- The fallback is "a last retry" after exhaustion, not a pre-first-event switch as in pi. There is no explicit "events already emitted" guard in the findings (unverified). The partial-output risk is bounded because WebSocket retry replays the whole request.

## Versus pi
- [[pi--http-transport-hardening|pi]] rotates pooled Codex sockets at a 55 min max age, below the 60 min limit. Codex instead reacts to the server's `websocket_connection_limit_reached` with a retry.
- Both make the WebSocket→HTTP fallback sticky per session.
- pi ties its header timeout to user `timeoutMs`. Codex uses a separate connect timeout (15 s) and one idle timeout applied to every receive and send.

## Failures
- [[stream-stall-without-header-timeout]]
- [[persistent-connection-lifetime-exceeded]]
- [[transport-fallback-after-partial-output]]
- [[server-side-state-missing-on-continuation]]
- [[prewarm-blocks-turn-start]]
- [[server-retry-advice-ignored]]
