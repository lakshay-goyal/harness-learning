---
type: failure
concepts: [errors-as-stream-events, auto-retry-backoff, subscription-usage-limits]
harnesses: [pi, codex]
---
**Symptom**
- Responses and Azure 5xx/429 errors were not auto-retried (#4232).
- Mistral `finish_reason:"error"` was not retried (#10487).
- Mid-stream "you can retry your request" errors ended runs (#6019).
- Bedrock throttling was read as overflow (#2699).

**Root cause** Both the agent-level retry classifier and overflow detection are regexes over `errorMessage` (`packages/ai/src/utils/retry.ts:30-107`; `overflow.ts`). An adapter's message wording is therefore part of the retry contract.

**Fix · [[pi]]**
- `52e13870a` 2026-05-18: prefix the HTTP status into Responses and Azure error messages (#4232).
- `0c7bb7c5c` 2026-09-14: prefix with the provider name (`"OpenAI API error"` vs `${provider} API error`) (`packages/ai/src/api/openai-responses.ts:34,221-229`).
- `7fb59f995` 2026-10-07: Mistral message `"Provider stopped with: error (server error)"`, so `server.?error` matches (`packages/ai/src/api/mistral-conversations.ts:922-938`; `retry.ts:47`) (#10487).
- `a3bf1eb39` 2026-03-30: Bedrock uses stable prefixes `Internal server error|Model stream error|Validation error|Throttling error|Service unavailable` (`bedrock-converse-stream.ts:365-410`). `errorMessage` is kept byte-identical and structured metadata sits alongside it "because isRetryableAssistantError matches text" (`:435`).
- `371adcf37` 2026-06-24: explicit "please retry" texts. The classifier moved into `packages/ai/src/utils/retry.ts` (#6019).
- The pattern list itself is covered in [[retry-classifier-regex-sprawl]].

**Fix · [[codex]]** — The inverse problem: deterministic failures were retried, or classified into the wrong bucket.
- Symptom:
  - Quota, credit and spend-limit errors were retried as transient (reports of "intermittent retry behaviors").
  - `slow_down` was treated as a terminal overload.
  - HTTP 429 quota errors were reported as retry-limit failures.
- `0c647bc566` 2025-11-06 (#6340): don't retry `insufficient_quota`.
- `102fc57e4a` 2026-09-10: HTTP quota codes → `QuotaExceeded`.
- `31ffe2bc9a` 2026-09-15: `slow_down` is retryable; `credit_balance_exhausted|organization_spend_limit_exceeded|project_spend_limit_exceeded` are terminal.
- Now: code tables classify by structured code, not text (`codex-rs/codex-api/src/sse/responses_error.rs:47-99`, and the 429 branch of `map_api_error_details` in `codex-rs/codex-api/src/api_bridge.rs`, with `insufficient_quota` at `:221`). Retryability is a property of the typed error (`CodexErr::retry_delay`, `codex-rs/protocol/src/error.rs:397-445`).
- See also [[hidden-sdk-retries-double-retry]] (`invalid_prompt`, bio-policy, invalid images).

**Lesson** — With string-classified retry, error formatting is part of the retry contract. Classifying by structured error code decouples adapters from the classifier, and must keep "retry later" separate from "plan or balance exhausted".

Related: [[errors-as-stream-events]] · [[auto-retry-backoff]] · [[retry-classifier-regex-sprawl]] · [[rate-limit-misread-as-overflow]] · [[foreign-sdk-error-shape-skips-retry]] · [[pi--errors-as-stream-events|pi]] · [[subscription-usage-limits]] · [[codex--errors-as-stream-events|codex]] · [[hidden-sdk-retries-double-retry]]
