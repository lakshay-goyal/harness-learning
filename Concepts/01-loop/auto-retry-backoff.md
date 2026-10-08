---
type: concept
stage: failure-handling
tier: must-have
aliases: [_prepareRetry, settings.retry, isRetryableAssistantError, RETRYABLE_PROVIDER_ERROR_PATTERN, retryProviderRequest, auto_retry_start, auto_retry_end, maxAgentDelayMs, maxRetryDelayMs, two-layer-retry, transient-error-classification, SessionRetry.policy, RETRY_MAX_RETRIES, RETRYABLE_MESSAGE_PATTERNS, RequestExecutor, retryStatusFailures, handle_response_stream_error, ResponsesStreamRetryState, "CodexErr::retry_delay", RetryOn, run_with_retry, stream_max_retries, request_max_retries, UnboundedConnectionRetries, "Reconnecting... n/m", ExhaustedResponseRetry, try_switch_fallback_transport]
harnesses: [pi, opencode, codex]
---
Harness-level retry of transient provider failures with capped exponential backoff; a classifier decides retryable vs terminal (quota, overflow, deterministic errors).

## Why
- Transient 429/5xx/network/early-EOF failures otherwise end headless runs ("waiting for a manual nudge") ([[retry-classifier-regex-sprawl]]).
- Hidden SDK retries under the agent loop double-retry, burn quota, and ignore cancellation ([[hidden-sdk-retries-double-retry]], [[retry-backoff-hygiene]]).
- Retry budgets scoped wrong exhaust across a tool-use run ([[retry-counter-accumulates-across-turn]]); deterministic failures retried pointlessly ([[deterministic-5xx-retried]]).
- Failed attempts must leave the model's context ([[abandoned-attempts-left-in-context]]); callers need to know a retry is pending ([[retry-wait-race-prompt-returns-early]]).
- Server retry advice ignored → hammering or retrying before the deadline ([[server-retry-advice-ignored]]); a semantic refusal retried verbatim reproduces itself ([[content-filter-retry-without-guidance]]).

## Design space
- **Layering**: SDK retries (hidden) · harness-owned single visible layer (pi after `8fb1e877c`) · two layers with escalation: adapter retries only pre-headers and throws long server delays to the visible layer (pi) · HTTP-request layer (5xx/transport only, 429 not retried) + loop-level stream layer that rebuilds the prompt from history each attempt (✔ codex).
- **Classifier**: regex over error text (pi; grows forever, couples adapters to wording) · structured error kinds/status codes (✔ codex: one `CodexErr::retry_delay(n)` returns terminal / advice-only / backoff) · header hints (`x-should-retry`, `Retry-After`).
- **Backoff**: `base·2^(n−1)` capped, no jitter (pi agent) · capped exponential with jitter (pi adapter) · uncapped 200 ms·2^(n−1) ±10 % bounded only by count (✔ codex) · server-specified delay with validation · server advice as monotonic deadline captured at header receipt, never extending the retry budget (✔ codex).
- **Budget scope**: per request, reset on success (pi `4f004adef`) · per sampling request/model step (✔ codex `ResponsesStreamRetryState` per `run_sampling_request`) · per user turn.
- **Exhaustion fallback**: fail (pi) · switch WebSocket→HTTPS once, reset counter, still honour server deadline (✔ codex).
- **Network outage**: bounded retries (pi) · opt-in unbounded connection retries 5 s→60 s "Reconnecting... waiting for network" outside the normal budget (✔ codex).
- **Prompt mutation on retry**: never (pi) · record model-specific content-filter guidance before retrying a content-filter stop (✔ codex).
- **Exclusions**: context overflow → compaction instead (pi) · quota/billing → terminal (pi).
- **Context hygiene**: delete failed attempt · hide via append-only edit (pi `context_edit`) · keep completed items and continue from partial progress (✔ codex).
- **Durability**: in-memory sleep (pi stable) · checkpointed retry phase with absolute deadline (pi-durable).
- **Layer placement**: session-level retry wrapping the whole step stream with SDK retries 0 (opencode legacy) · transport-only retry before the first byte, none at runner level (opencode v2).
- **Attempt bound**: none (opencode legacy until `c78986831c`, ~10 months) · 5 attempts with positive 0–25 % jitter (opencode legacy HEAD) · 2 attempts ±20 % (opencode v2).
- **Server delay**: honor any `retry-after` up to int32 ms (opencode legacy) · clamp to 10 s (opencode v2).
- **Classifier order**: structured mid-stream error codes, then status ≥ 500, then 7 regexes over message and body (opencode legacy).

## Implementations
- [[pi--auto-retry-backoff|pi]] — `settings.retry` 3 × 2s/4s/8s capped 60 s; regex classifier in pi-ai; abortable adapter retry with 60 s server-delay escalation; failed attempts omitted via `context_edit`.
- [[codex--auto-retry-backoff|codex]] — HTTP layer (4 retries, 5xx/transport) + stream layer (5 retries, cap 100) decided by `CodexErr::retry_delay`; 200 ms ×2 ±10 % backoff; Retry-After deadlines; WS→HTTPS fallback; unbounded network wait; cancellable by interrupt and steer.
- [[opencode--auto-retry-backoff|opencode]] — legacy `SessionRetry.policy` 2 s·2^n + jitter, cap 30 s, 5 attempts (unbounded until 2026-08); v2 `RequestExecutor` 2 retries pre-output only.

## Failures
- [[retry-wait-race-prompt-returns-early]]
- [[retry-counter-accumulates-across-turn]]
- [[retry-classifier-regex-sprawl]]
- [[retry-backoff-hygiene]]
- [[hidden-sdk-retries-double-retry]]
- [[deterministic-5xx-retried]]
- [[error-text-breaks-retry-classification]] (02-model-interface)
- [[foreign-sdk-error-shape-skips-retry]] (02-model-interface)
- [[abandoned-attempts-left-in-context]]
- [[server-retry-advice-ignored]]
- [[content-filter-retry-without-guidance]]
- [[transport-fallback-after-partial-output]] (02-model-interface)
- [[summary-call-not-retried]] (05-context) — A transient stream drop (terminated, socket close) during the summarization call failed the whole compaction…

## Related
[[terminal-event-required]] · [[errors-as-stream-events]] · [[context-overflow-detection]] · [[overflow-recovery]] · [[context-edit-overlay]] · [[run-settlement]] · [[abort-propagation]] · [[http-transport-hardening]] · [[virtual-model-router]] · [[subscription-usage-limits]] · [[steering-queue]]
