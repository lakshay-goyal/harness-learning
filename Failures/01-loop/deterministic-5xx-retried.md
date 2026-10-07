---
type: failure
concepts: [auto-retry-backoff, structured-classifier-api]
harnesses: [pi, opencode, codex]
---
**Symptom** OpenAI Decisions requests with huge inputs (> ~600K tokens) got an HTML 504, and retrying hit it again, wasting time and money.

**Root cause** The Cloudflare edge in front of api.openai.com returns 504 when a request runs past ~5 s. That makes the 504 deterministic for a given input, not transient.

**Fix · [[pi]]** `ce8972a0e` 2026-10-07: `noRetryStatuses: [504]` plus an explanatory error message (`packages/ai/src/api/openai-decisions.ts:146-158`). The per-call opt-out hook is `noRetryStatuses` (`packages/ai/src/utils/provider-retry.ts:7-8,121`). Classifier HTTP uses `retryProviderRequest` with `maxRetries ?? 2` and a fresh timeout per attempt (`packages/ai/src/api/classifier-shared.ts:60-104`).

**Fix · [[codex]]** (variant: deterministic non-5xx failures retried)
- Symptom: `insufficient_quota` was retried "multiple times in the agent loop", causing "intermittent retry behaviors" (`0c647bc566` 2025-11-06 "Don't retry insufficient_quota errors (#6340)"); invalid tool images were replaced with "Invalid image" text and the request retried every turn (removed `8431dc590a` 2026-07-20 "Stop retrying turns with invalid tool images").
- HEAD: terminal error classes return `None` from `CodexErr::retry_delay` (QuotaExceeded, UsageNotIncluded, InvalidRequest, InvalidImageRequest, policy violations…) (`codex-rs/protocol/src/error.rs:397-424`); invalid image ⇒ bad-request error event "Invalid image in your last message..." without touching history (`codex-rs/core/src/session/turn.rs:793-816`).
**Fix · [[opencode]]** `62f38087b8` 2026-02-08 (same class: deterministic failure retried): OpenAI-Responses-style mid-stream error events (`insufficient_quota`, `usage_not_included`, `invalid_prompt`, `context_length_exceeded`) were unparsed, became retryable unknown errors and were **retried forever**; `parseStreamError` now maps them to non-retryable or overflow (`packages/opencode/src/provider/error.ts:102-139`).

**Lesson** — Some failures (deterministic 5xx, quota, invalid input) recur for the same request; classify them terminal on a typed error rather than per call site, and explain the cause.
Related: [[structured-classifier-api]] · [[auto-retry-backoff]] · [[pi--structured-classifier-api|pi]] · [[opencode--auto-retry-backoff|opencode]]

Related: [[structured-classifier-api]] · [[auto-retry-backoff]] · [[pi--structured-classifier-api|pi]] · [[codex--auto-retry-backoff|codex]]
