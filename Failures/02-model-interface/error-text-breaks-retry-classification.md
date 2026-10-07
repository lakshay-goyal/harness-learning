---
type: failure
concepts: [errors-as-stream-events, auto-retry-backoff]
harnesses: [pi, opencode]
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

**Fix · [[opencode]]** `027d43b5ea` 2025-12-01: provider error bodies concatenated into the message broke retry detection; messages normalized. `61aefc0759` 2026-08-05: the regex list also matches the response body, not only the message (`packages/opencode/src/session/retry.ts:93-94`).

**Lesson** With string-classified retry, error formatting is part of the retry contract. Structured error kinds would decouple adapters from the classifier.

Related: [[errors-as-stream-events]] · [[auto-retry-backoff]] · [[retry-classifier-regex-sprawl]] · [[rate-limit-misread-as-overflow]] · [[foreign-sdk-error-shape-skips-retry]] · [[pi--errors-as-stream-events|pi]] · [[opencode--auto-retry-backoff|opencode]]
