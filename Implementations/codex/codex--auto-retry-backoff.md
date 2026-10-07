---
type: implementation
harness: codex
concept: auto-retry-backoff
commit: 622e9e3696
files: [codex-rs/codex-client/src/retry.rs:84, codex-rs/model-provider-info/src/lib.rs:67, codex-rs/model-provider-info/src/lib.rs:454, codex-rs/core/src/responses_retry.rs:57, codex-rs/protocol/src/error.rs:397, codex-rs/async-utils/src/backoff.rs:7, codex-rs/core/src/session/turn.rs:1638, codex-rs/codex-api/src/api_bridge.rs:149]
---
[[auto-retry-backoff]] in [[codex]].

## Mechanism
1. **HTTP-request layer** (`run_with_retry`, `codex-rs/codex-client/src/retry.rs:84-118`): attempts `0..=max_attempts`; `RetryOn::should_retry` retries 429 only if `retry_429`, any 5xx if `retry_5xx`, Timeout/Connection/Network if `retry_transport`; never Build/RetryLimit/Policy/ResponseTooLarge (`:23-40`). OpenAI-style provider: `retry_429: false, retry_5xx: true, retry_transport: true, base_delay 200ms, max_attempts = request_max_retries` (`codex-rs/model-provider-info/src/lib.rs:454-460`) → 429 surfaces to the stream layer. Same backoff shape (`codex-rs/codex-client/src/retry.rs:43-52`).
2. **Stream/turn layer** `handle_response_stream_error` (`codex-rs/core/src/responses_retry.rs:57-176`), shared by sampling and remote compaction v2. `run_sampling_request` rebuilds the prompt from current history each attempt (`codex-rs/core/src/session/turn.rs:1638-1729`); retry state per call (`:1634`) → no accumulation across steps. `ContextWindowExceeded` and `UsageLimitReached` bypass retry (`:1688-1701`).
3. **Classifier** `CodexErr::retry_delay(retry_count)` (`codex-rs/protocol/src/error.rs:397-446`): `None` (terminal) for TurnAborted, Interrupted, UsageNotIncluded, QuotaExceeded, UsageLimitReached, InvalidRequest, InvalidPrompt, InvalidImageRequest, ContextWindowExceeded, RefreshTokenFailed, policy violations (Cyber/Bio/Misalignment), FlexUnavailable, SessionBudgetExceeded, Fatal, sandbox errors; `ServerOverloaded` and `RetryLimit` retry ONLY with server Retry-After advice (`:424-426`); Stream, ContentFilter, RateLimitExceeded, Timeout, RequestTimeout, UnexpectedStatus, ResponseStreamFailed, ConnectionFailed, InternalServerError, Io, Json retry with advice or local backoff (`:427-442`). Error-code tables: `codex-rs/codex-api/src/sse/responses_error.rs:47-99`; `INVALID_PROMPT_ERROR_CODE`, `insufficient_quota` (`codex-rs/codex-api/src/api_bridge.rs:149-167`, `:221`) → [[subscription-usage-limits]].
4. **Backoff**: 200 ms × 2^(n−1), ±10 % jitter, attempts 0 and 1 both 200 ms, no cap on the delay itself (5 stream retries → max ~3.2 s) (`codex-rs/async-utils/src/backoff.rs:7-17`).
5. **Server advice**: `Retry-After` (seconds or HTTP-date) parsed into a monotonic deadline captured when headers arrive; retry sleeps until it; expired advice = zero delay, never falls back to local backoff (`codex-rs/protocol/src/error.rs:449-458`; `codex-rs/http-client/src/retry_after.rs`); advice never extends the configured budget (`codex-rs/core/src/responses_retry.rs:1-3`); nested `response.error.headers` in streamed failures preferred over message text → [[server-retry-advice-ignored]].
6. **Exhaustion**: first try WebSocket→HTTPS fallback (resets counter to 0, still waits for server deadline, Warning "Falling back from WebSockets to HTTPS transport.") (`codex-rs/core/src/responses_retry.rs:120-139`); else store `ExhaustedResponseRetry{turn_id, retry_at}` in thread extension data so a reused Guardian reviewer session doesn't apply stale advice (`:48-53`, `:169-175`) → [[http-transport-hardening]].
7. **Unbounded connection retries** (`Feature::UnboundedConnectionRetries`; user, non-internal sampling on non-Bedrock providers): `ConnectionFailed` retries forever, 5 s doubling to 60 s, "Reconnecting... waiting for network", not consuming the stream budget (`codex-rs/core/src/responses_retry.rs:23-24`, `:93-118`).
8. **Content filter**: before the retry decision a model-specific content-filter guidance fragment (`ModelMessages.content_filter_guidance`, ≤512 bytes, `codex-rs/protocol/src/openai_models.rs:543-546`) is recorded into history, then retried (`codex-rs/core/src/responses_retry.rs:67-82`) → [[content-filter-retry-without-guidance]].
9. **UI noise**: "Reconnecting... {n}/{max}" stream-error event; first WebSocket retry hidden in release builds (`codex-rs/core/src/responses_retry.rs:145-159`).
10. **Cancellation**: retry sleep wrapped `.or_cancel(&preempt).or_cancel(&cancellation_token)` — interrupt and steer-preempt both cut it (`codex-rs/core/src/session/turn.rs:1705-1729`) → [[retry-backoff-hygiene]].
11. **Telemetry**: `record_retry!` emits `codex.retry` trace event with attempt, delay_ms, layer (http|stream), operation (request|sampling|remote_compaction_v2) (`codex-rs/codex-client/src/retry.rs:62-82`).
12. **Partial progress**: items completed in a failed attempt stay in history and in-flight tools are drained; the retry continues from there (`codex-rs/core/src/session/turn.rs:3156-3165`, `:1641-1649`).

## Constants
| name | value | path:line |
|---|---|---|
| stream (loop-level) retries default | `DEFAULT_STREAM_MAX_RETRIES` = 5 | `codex-rs/model-provider-info/src/lib.rs:67` |
| HTTP request retries default | `DEFAULT_REQUEST_MAX_RETRIES` = 4 | `codex-rs/model-provider-info/src/lib.rs:68` |
| user retry config hard cap | `MAX_STREAM_MAX_RETRIES` = `MAX_REQUEST_MAX_RETRIES` = 100 | `codex-rs/model-provider-info/src/lib.rs:72-75` |
| backoff initial / factor / jitter | `INITIAL_DELAY_MS` 200 / `BACKOFF_FACTOR` 2.0 / ±10 %, uncapped | `codex-rs/async-utils/src/backoff.rs:7-17` |
| HTTP retry base delay / `retry_429` | 200 ms / false | `codex-rs/model-provider-info/src/lib.rs:456-457` |
| unbounded connection-retry delay | `INITIAL_CONNECTION_RETRY_DELAY` 5 s → ×2 → `MAX_CONNECTION_RETRY_DELAY` 60 s | `codex-rs/core/src/responses_retry.rs:23-24` |
| remote compaction stream retries | min(provider, 2) | `codex-rs/core/src/compact_remote_v2.rs:77` |
| content-filter guidance cap | ≤512 bytes | `codex-rs/protocol/src/openai_models.rs:543-546` |

## Evolution
- 2025-04-17 `693a6f96cf` (TS era) retry regex missed "Please try again in 3.965s" → [[retry-classifier-regex-sprawl]]; 2025-05-11 `e307d007aa` retry server_error without status code.
- 2025-07-18 `9846adeabf` provider-level retry/idle settings; 2025-08-07 `548466df09` "[client] Tune retries and backoff (#1956)": stream retries 10→5 ("10 is a bit excessive"), factor 1.3→2.0.
- 2025-08-13 `41eb59a07d` server delay parsed from message; 2025-08-25 `d32e4f25cf` retry config capped at 100.
- 2025-11-02 `d9118c04bf` fuzzy "try again in" regex (Azure); 2025-11-06 `0c647bc566` "Don't retry insufficient_quota errors (#6340)"; 2025-11-25 `4502b1b263` 429 moved out of HTTP layer.
- 2026-01-16 `93a5e0fe1c` `invalid_prompt` not retried; 2026-01-29 `3b1cddf001` / 2026-02-07 `6d08298f4e` WebSocket→HTTP fallback (426); 2026-02-11 `d391f3e2f9` hide first WebSocket retry.
- 2026-05-26 `04a8580f33` "centralize Responses retry policy (#24131)"; 2026-07-20 `8431dc590a` invalid tool images no longer retried.
- 2026-08-07 `5a0d0929e2` unbounded connection retries (#37485), configurable `da898490fc` 2026-08-14; 2026-08-13 `1b4ea8b3be` retry telemetry.
- 2026-09-10 `102fc57e4a` HTTP quota codes → QuotaExceeded; 2026-09-15 `31ffe2bc9a` `slow_down` retryable, spend-limit codes terminal; 2026-09-17 `fa8cf44985` bio-policy terminal; 2026-09-18 `d5b29951ac` "Centralize retry decisions and delays in `CodexErr` (#46540)" (replaced `is_retryable()`).
- 2026-09-23 `9d8de19674` Retry-After deadlines (#47641); 2026-09-29 `a9118edae8` content-filter guidance in shared handler; 2026-09-30 `6ba4bf9e64` overload/RetryLimit retry only with advice (#49441); 2026-10-02 `44dd77b71e` Retry-After in failed Responses events; 2026-10-06 `f6cf05af1d` Retry-After in WebSocket errors.

## Quirks
- A length stop (`response.incomplete` reason ≠ interrupted/content_filter) is a retryable Stream error, so it is retried rather than continued ([[truncated-tool-call-guard]]).
- No overflow-retry: `ContextWindowExceeded` fails the turn; next turn compacts first ([[overflow-recovery]]).
- Retry with a prompt mutation (content-filter guidance) — unusual.

## Versus pi
- [[pi--auto-retry-backoff]]: pi = regex classifier + 3 retries 2/4/8 s capped 60 s, SDK retries disabled; codex = typed classifier on `CodexErr` with terminal / advice-only / backoff split, 5 + 4 retries in two layers, 200 ms base uncapped, monotonic server deadlines, transport fallback as last retry, unbounded network-wait mode.
- Both scope budgets per request/step (contrast pi [[retry-counter-accumulates-across-turn]] fix).
