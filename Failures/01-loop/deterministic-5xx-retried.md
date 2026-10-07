---
type: failure
concepts: [auto-retry-backoff, structured-classifier-api]
harnesses: [pi, opencode]
---
**Symptom** OpenAI Decisions requests with huge inputs (> ~600K tokens) got an HTML 504, and retrying hit it again, wasting time and money.

**Root cause** The Cloudflare edge in front of api.openai.com returns 504 when a request runs past ~5 s. That makes the 504 deterministic for a given input, not transient.

**Fix · [[pi]]** `ce8972a0e` 2026-10-07: `noRetryStatuses: [504]` plus an explanatory error message (`packages/ai/src/api/openai-decisions.ts:146-158`). The per-call opt-out hook is `noRetryStatuses` (`packages/ai/src/utils/provider-retry.ts:7-8,121`). Classifier HTTP uses `retryProviderRequest` with `maxRetries ?? 2` and a fresh timeout per attempt (`packages/ai/src/api/classifier-shared.ts:60-104`).

**Fix · [[opencode]]** `62f38087b8` 2026-02-08 (same class: deterministic failure retried): OpenAI-Responses-style mid-stream error events (`insufficient_quota`, `usage_not_included`, `invalid_prompt`, `context_length_exceeded`) were unparsed, became retryable unknown errors and were **retried forever**; `parseStreamError` now maps them to non-retryable or overflow (`packages/opencode/src/provider/error.ts:102-139`).

**Lesson** Some 5xx errors are deterministic for a given input. Exclude them from retry and explain the cause.

Related: [[structured-classifier-api]] · [[auto-retry-backoff]] · [[pi--structured-classifier-api|pi]] · [[opencode--auto-retry-backoff|opencode]]
