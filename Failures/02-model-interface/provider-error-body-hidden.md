---
type: failure
concepts: [errors-as-stream-events]
harnesses: [pi]
---
**Symptom** Proxy and gateway errors showed only `"403 status code (no body)"` or `"Unknown: UnknownError"`, especially a Bedrock gateway 403 (#5763). After enrichment was added, AWS errors showed `{"_events":...}` garbage that had *replaced* the real message.

**Root cause**
- SDKs do not fold the HTTP body into `message`.
- The first normalizer stringified the AWS SDK `$response.body`, which is a class instance or stream, not a plain object.

**Fix · [[pi]]**
- `62fad94f1` 2026-06-17: `normalizeProviderError` probes the status as `statusCode` → `status` → `$metadata.httpStatusCode` → `$response.statusCode`. It probes the body as a `body` string → `error` object → `$response.body`. `formatProviderError` produces `"<prefix> (<status>): <body>"` only when the message lacks the body (`packages/ai/src/utils/error-body.ts:56-92,119-135`). Body capped at `MAX_PROVIDER_ERROR_BODY_CHARS = 4000` with `... [truncated N chars]` (`:16,137-140`) (#5763).
- `4523528b2` 2026-07-31: only plain-prototype objects count as bodies (`error-body.ts:98-117`).
- `0b6c95dda` 2026-06-09: AWS docs hint for Bedrock data-retention errors.
- `70bbe47a9` 2026-07-30: structured `bedrock_response_failure {status, errorCode, requestId}` diagnostic. Values over 200 chars are dropped, not truncated ("a truncated request id is not a request id"). `errorMessage` is kept byte-identical (`bedrock-converse-stream.ts:412-458`).
- Observation: the Anthropic adapter does not use `normalizeProviderError`.

**Lesson** Always surface the HTTP status and body. When enriching an error, never discard the original message in favor of a lower-confidence source.

Related: [[errors-as-stream-events]] · [[error-text-breaks-retry-classification]] · [[pi--errors-as-stream-events|pi]]
