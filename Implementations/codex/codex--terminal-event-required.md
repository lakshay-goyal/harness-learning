---
type: implementation
harness: codex
concept: terminal-event-required
commit: 622e9e3696
files: [codex-rs/core/src/session/turn.rs:2658, codex-rs/codex-api/src/sse/responses.rs:536, codex-rs/codex-api/src/sse/responses.rs:557, codex-rs/protocol/src/error.rs:432, codex-rs/model-provider-info/src/lib.rs:66, codex-rs/codex-api/src/endpoint/responses_websocket.rs:897, codex-rs/core/src/client.rs:2366]
---
[[terminal-event-required]] in [[codex]].

## Mechanism
- **Loop check**: stream `None` before `response.completed` ⇒ `CodexErr::Stream("stream closed before response.completed")` (`codex-rs/core/src/session/turn.rs:2658-2663`); retryable via `retry_delay` (Stream is in the backoff arm, `codex-rs/protocol/src/error.rs:432-442`) → [[auto-retry-backoff]].
- **Parser check**: SSE parser emits the same error when the body ends without `response.completed` (`codex-rs/codex-api/src/sse/responses.rs:557-563`); a dropped stream before the terminal event has its own `STREAM_DROPPED_REASON` (`codex-rs/core/src/client.rs:2366`).
- **Idle timeout**: SSE read loop wraps each `next()` in `timeout(idle_timeout)` → `Stream("idle timeout waiting for SSE")`, retryable (`codex-rs/codex-api/src/sse/responses.rs:536-569`); same timeout bounds WebSocket receive and, since `35aaa5d9fc`, WebSocket send (`codex-rs/codex-api/src/endpoint/responses_websocket.rs:693`, `:897-903`) → [[http-transport-hardening]].
- **Unparseable events** logged and skipped, not fatal (`codex-rs/codex-api/src/sse/responses.rs:573-585`).
- **Completed handling**: `ResponseEvent::Completed` records usage, applies token budget, honours `end_turn` (`codex-rs/core/src/session/turn.rs:2947-2995`).
- **Partial output**: output items recorded as each `OutputItemDone` arrives and in-flight tools drained even after a stream error, so the retry continues from history (`codex-rs/core/src/stream_events_utils.rs:347`, `codex-rs/core/src/session/turn.rs:3156-3165`).

## Constants
| name | value | path:line |
|---|---|---|
| stream idle timeout | `DEFAULT_STREAM_IDLE_TIMEOUT_MS` = 300 000 | `codex-rs/model-provider-info/src/lib.rs:66` |
| WebSocket connect timeout | `DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS` = 15 000 | `codex-rs/model-provider-info/src/lib.rs:71` |
| response stream channel capacity | 1600 events | `codex-rs/core/src/client.rs:2365` |

## Evolution
- 2025-07-18 `9846adeabf` stream idle timeout default.
- 2026-03-17 `6ea041032b` 15 s WebSocket connect timeout (#14838).
- 2026-05-01 `35aaa5d9fc` "Bound websocket request sends with idle timeout" (#20751) — sessions "recover only after a long quiet period when the server had already logged the websocket as disconnected".

## Versus pi
- [[pi--terminal-event-required]]: pi enforces a terminal marker per adapter (many wires, compat flag for servers without one); codex has one wire (Responses over SSE/WebSocket) and checks `response.completed` twice (parser + loop). Both classify as retryable.
