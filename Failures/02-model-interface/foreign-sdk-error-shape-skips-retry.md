---
type: failure
concepts: [errors-as-stream-events, auto-retry-backoff]
harnesses: [pi, opencode]
---
**Symptom** On Google/Vertex, a transient 429 or 5xx before the first token became a terminal errored message, with no retry (#7471).

**Root cause** The Google SDK `ApiError` has a `status` but no `headers`, so the shared `isProviderError` guard in `retryProviderRequest` rejected it.

**Fix · [[pi]]** `b9d360a2c` 2026-08-03: `retryGoogleRequest` adds `headers = undefined` and delegates to the shared retry (`packages/ai/src/api/google-shared.ts:494-515`, guard `:503-505`) (#7471). Consequences:
- Google `retry-after` and the body `RetryInfo.retryDelay` are never read, so backoff is pure exponential `min(0.5·2^i, 8)s` (`packages/ai/src/utils/provider-retry.ts:53-69`). Body RetryInfo parsing existed only in the Gemini CLI provider removed in `fe66edd94`.
- Only the initial call is retried. Mid-stream failures are not.

**Fix · [[opencode]]** `4ca809ef4e` 2026-04-15: 5xx errors were not retried when the provider SDK left `isRetryable` unset; status ≥ 500 is now always retried (`packages/opencode/src/session/retry.ts:89-96`).

**Lesson** Normalize foreign SDK error objects into the shared retry shape at the adapter boundary.

Related: [[errors-as-stream-events]] · [[auto-retry-backoff]] · [[error-text-breaks-retry-classification]] · [[retry-backoff-hygiene]] · [[pi--errors-as-stream-events|pi]] · see also [[retry-classifier-regex-sprawl]] (01-loop view of the same retry-miss class) · [[opencode--auto-retry-backoff|opencode]]
