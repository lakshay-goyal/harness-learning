---
type: implementation
harness: opencode
concept: auto-retry-backoff
commit: ecc4916b5a
files: [packages/opencode/src/session/retry.ts:26-41, packages/opencode/src/session/retry.ts:80-155, packages/opencode/src/session/retry.ts:183-207, packages/opencode/src/session/processor.ts:674-688, packages/opencode/src/session/llm.ts:323, packages/opencode/src/provider/error.ts:102-139, packages/llm/src/route/executor.ts:35-38, packages/llm/src/route/executor.ts:91, packages/llm/src/route/executor.ts:344-364]
---
[[auto-retry-backoff]] in [[opencode]].

## Mechanism

### Legacy runtime — session-level retry around the whole step stream
- `Effect.retry(SessionRetry.policy(...))` wraps the stream effect inside `SessionProcessor.process` (`packages/opencode/src/session/processor.ts:674-688`). AI SDK retries off: `maxRetries: input.retries ?? 0` (`packages/opencode/src/session/llm.ts:323`) → one visible layer (title side call passes `retries: 2`, `packages/opencode/src/session/prompt.ts:234`).
- Classifier `retryable()` (`packages/opencode/src/session/retry.ts:85-155`):
  - never `ContextOverflowError` (goes to compaction);
  - `APIError` retried if `isRetryable` OR status ≥ 500 OR message/body matches `RETRYABLE_MESSAGE_PATTERNS` (7 regexes: 429/5xx codes, rate limit, overloaded, network/socket/DNS errnos, timeouts, "try again", capacity, `retry.ts:33-41`);
  - non-API errors on `too_many_requests` / `exhausted` / `unavailable` / pattern match;
  - body `FreeUsageLimitError` / `GoUsageLimitError` → product upsell action in the retry status.
- Mid-stream Responses-style error events parsed by code (`parseStreamError`): `context_length_exceeded` → overflow; `insufficient_quota`, `usage_not_included`, `invalid_prompt` → non-retryable; `server_error` → retryable (`packages/opencode/src/provider/error.ts:102-139`).
- Delay (`retry.ts:47-83`): `retry-after-ms`, else `retry-after` seconds or HTTP-date — capped only at int32 ms; else `ceil(2000·2^(n−1)·(1 + 0.25·rand))` capped at 30 s.
- Policy: `Schedule.fromStepWithMetadata`; stop when not retryable or `attempt > RETRY_MAX_RETRIES` (`retry.ts:183-207`). Status `{type:"retry", attempt, message, action, next}` shown to UI.
- Retried attempts reuse the same assistant message; only in-memory cursors reset, so parts persisted by the failed attempt stay (`processor.ts:649-652`) → [[abandoned-attempts-left-in-context]].

### v2 runtime — transport-level only
- No runner-level retry: a provider error ends the drain (`packages/core/src/session/runner/llm.ts:55` unchecked "Bound provider retries"; `session.next.retried` projector commented out, `packages/core/src/session/projector.ts:392`).
- `RequestExecutor.retryStatusFailures`: retries only `LLMError.retryable` failures before any stream output, ≤ 2 times, delay `min(retryAfterMs, 10 s)` or `500·2^n` with ±20 % jitter (`packages/llm/src/route/executor.ts:344-364`). Retryable = 429/503/504/529 or any ≥ 500 (`ProviderInternalReason`), 429 unless body says quota (`executor.ts:91,242-273`).
- Design rule: "Retry only before observable output … Never silently retry after ambiguous tool execution" (`packages/llm/DESIGN.md:704-714`).
- Mid-stream `retryable` flags (Bedrock throttling, Anthropic `overloaded_error`) are never read by `packages/core/src` → they end the drain.

## Constants
| name | value | path:line |
|---|---|---|
| `RETRY_INITIAL_DELAY` | 2000 ms | `packages/opencode/src/session/retry.ts:26` |
| `RETRY_BACKOFF_FACTOR` | 2 | `packages/opencode/src/session/retry.ts:27` |
| `RETRY_JITTER_FACTOR` | 0.25 (positive only) | `packages/opencode/src/session/retry.ts:28` |
| `RETRY_MAX_DELAY_NO_HEADERS` | 30 000 ms | `packages/opencode/src/session/retry.ts:29` |
| `RETRY_MAX_DELAY` | 2 147 483 647 ms (int32, ≈24.8 days) | `packages/opencode/src/session/retry.ts:30` |
| `RETRY_MAX_RETRIES` | 5 | `packages/opencode/src/session/retry.ts:31` |
| v2 `MAX_RETRIES` / `BASE_DELAY_MS` / `MAX_DELAY_MS` | 2 / 500 / 10 000 | `packages/llm/src/route/executor.ts:36-38` |

## Evolution
- 2025-10-22 `7c7ebb0a9d` "retry parts" — session retry introduced, **no attempt cap**.
- 2025-12-01 `027d43b5ea` provider error bodies concatenated into the message broke detection.
- 2025-12-31 `9a1dc1ffe4` clamp delay to int32 (huge `retry-after` → `TimeoutOverflowWarning` → fired immediately → hot loop).
- 2026-01-27 `837037cd04` OpenAI 404s retried.
- 2026-02-08 `62f38087b8` parse mid-stream errors: fatal codes were retried forever.
- 2026-04-15 `4ca809ef4e` 5xx retried even when SDK left `isRetryable` unset.
- 2026-08-05 `61aefc0759` regex list over message + body; 2026-08-20 `71d08e94d5` xAI capacity; 2026-08-21 `40282c1d4d` network-error variants, `e0b9e68a68` raw `network_error` finish.
- 2026-08-11 `c78986831c` "cap session retries with jitter" — **unbounded for ~10 months before**.

## Quirks / drift
- Honors any server `retry-after` up to int32 ms (legacy) vs 10 s clamp (v2).
- Legacy classifier is text-first with a structured-code side path; v2 classifies by typed reason but drops mid-stream retryability.

Failures: [[retry-backoff-hygiene]] · [[retry-classifier-regex-sprawl]] · [[deterministic-5xx-retried]] · [[foreign-sdk-error-shape-skips-retry]] · [[error-text-breaks-retry-classification]] · [[abandoned-attempts-left-in-context]].

Contrast: [[pi--auto-retry-backoff|pi]] caps at 3 attempts with a 60 s server-delay escalation from day one; opencode legacy ran unbounded until `c78986831c` and v2 moved retry down into the transport.
