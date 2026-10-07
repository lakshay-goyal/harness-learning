---
type: implementation
harness: codex
concept: errors-as-stream-events
commit: 622e9e3696
files: [codex-rs/codex-api/src/sse/responses.rs:550-569, codex-rs/codex-api/src/sse/responses_error.rs:47-142, codex-rs/codex-api/src/api_bridge.rs:149-221, codex-rs/protocol/src/error.rs:397-458, codex-rs/response-debug-context/src/lib.rs:5-60, codex-rs/core/src/client.rs:2365-2366]
---
[[errors-as-stream-events]] in [[codex]].

## Mechanism
- **The stream ends with an error item.** The SSE task sends `Err(ApiError)` as the last stream item and closes (`codex-rs/codex-api/src/sse/responses.rs:550-569`). If the response stream is dropped before the terminal event, it fails with `STREAM_DROPPED_REASON` = "response stream dropped before provider terminal event" (`codex-rs/core/src/client.rs:2366`). The channel holds `RESPONSE_STREAM_CHANNEL_CAPACITY` = 1600 events (`codex-rs/core/src/client.rs:2365`).
- **Two-stage typed normalization**, not text:
  1. Wire error → `ApiError` (`codex-rs/codex-api/src/error.rs`, `codex-rs/codex-api/src/sse/responses_error.rs`).
  2. `ApiError` → `CodexErr` with typed `CodexErrorDetails` and an optional retry deadline (`map_api_error` in `codex-rs/codex-api/src/api_bridge.rs`).
- **Retry decision lives on the error type.** `CodexErr::retry_delay(retry_count)` returns `None` for terminal errors, a server-advised delay, or local backoff (`codex-rs/protocol/src/error.rs:397-445`). Server advice is a monotonic deadline (`codex-rs/protocol/src/error.rs:449-458`). See [[auto-retry-backoff]].
- **`response.failed` code table** (`codex-rs/codex-api/src/sse/responses_error.rs:47-99`):
  | code | CodexErr | retry |
  |---|---|---|
  | `context_length_exceeded` | ContextWindowExceeded | terminal |
  | `insufficient_quota`, `credit_balance_exhausted`, `organization_spend_limit_exceeded`, `project_spend_limit_exceeded` | QuotaExceeded | terminal |
  | `usage_not_included` | UsageNotIncluded | terminal |
  | `cyber_policy`, `bio_policy`, `misalignment_policy_violation` | distinct policy errors with fallback text | terminal |
  | `invalid_prompt` | InvalidPrompt | terminal |
  | `server_is_overloaded` | ServerOverloaded | only with server advice |
  | `rate_limit_exceeded`, `slow_down` | RateLimitExceeded | delay from nested `Retry-After`, else the message regex `(?i)try again in\s*(\d+(?:\.\d+)?)\s*(s\|ms\|seconds?)` (`codex-rs/codex-api/src/sse/responses_error.rs:105-142`) |
  | other | generic Retryable | backoff |
- **HTTP status table** (`map_api_error_details`, Transport::Http branch of `codex-rs/codex-api/src/api_bridge.rs`; `INVALID_PROMPT_ERROR_CODE` at `codex-rs/codex-api/src/api_bridge.rs:149-167`, `insufficient_quota` at `codex-rs/codex-api/src/api_bridge.rs:221`):
  - 503 with the `server_is_overloaded` or `slow_down` codes.
  - 400 or 403 misalignment.
  - 400: policy or invalid_prompt. "image data … not a valid image" → InvalidImageRequest. Otherwise InvalidRequest.
  - 500 → InternalServerError.
  - 429:
    - flex-unavailable;
    - `usage_limit_reached` → UsageLimitReached with plan_type, resets_at, limit_window_minutes, a rate-limit snapshot and a promo message ([[subscription-usage-limits]]);
    - `usage_not_included`;
    - quota codes → QuotaExceeded;
    - anything else → RetryLimit, retried only with server advice.
  - Other statuses → UnexpectedStatus, carrying `cf-ray`, `x-request-id`/`x-oai-request-id`, `x-openai-authorization-error` and the `x-error-json` code.
  - A 403 Cloudflare HTML page → friendly `CLOUDFLARE_BLOCKED_MESSAGE`.
- **Transport errors:**
  - Timeout → RequestTimeout.
  - Connection → ConnectionFailed, with the URL redacted (`5a0d0929e2`).
  - Policy (network-policy denial) → Fatal.
  - ResponseTooLarge → InvalidRequest.
- **Debug context:** `x-request-id`, `x-oai-request-id`, `cf-ray`, `x-openai-authorization-error` and `x-error-json` are extracted for errors and bug reports (`codex-rs/response-debug-context/src/lib.rs:5-60`).
- **Error diagnostics are structural.** Catalog decode errors report only the JSON error category, line, column and byte count; SSE parse errors log the same bounded fields (`codex-rs/codex-api/src/sse/responses.rs:573-585`). See [[error-diagnostics-echo-payload]].

## Evolution
- `90ef94d3b3` 2025-10-04 (#4675): surface the context-window error to the client.
- `0c647bc566` 2025-11-06 (#6340): don't retry `insufficient_quota`.
- `93a5e0fe1c` 2026-01-16: `invalid_prompt` became terminal (it had been retried "despite the prompt being marked as disallowed").
- `5a0d0929e2` 2026-08-07: connection errors classified without leaking URLs.
- `102fc57e4a` 2026-09-10: HTTP quota codes → QuotaExceeded.
- `31ffe2bc9a` 2026-09-15: `slow_down` retryable; credit/spend codes terminal.
- `977193486d` 2026-09-16: catalog decode errors report category/line/column/size only; a catalog deadline is classified as RequestTimeout.
- `fa8cf44985` 2026-09-17: bio-policy errors non-retryable.
- `d5b29951ac` 2026-09-18 (#46540): `is_retryable()` → `retry_delay(retry_count)`.
- `44dd77b71e` 2026-10-02: nested headers in `response.failed` preferred over message text.

## Versus pi
- [[pi--errors-as-stream-events|pi]] classifies by regex over `errorMessage` text. Codex classifies by structured provider *codes* and HTTP status into a typed enum. Only the rate-limit delay still has a message-regex fallback.
- Partial content is not carried in the error. Completed output items are recorded one by one, including on abort: `f59978ed3d` 2025-10-23, see [[turn-items-lost-on-abort]].

## Failures
- [[error-text-breaks-retry-classification]]
- [[error-diagnostics-echo-payload]]
- [[stop-reason-mapping-gaps]]
- [[retry-classifier-regex-sprawl]]
