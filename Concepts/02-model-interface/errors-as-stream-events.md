---
type: concept
stage: failure-handling
tier: candidate
aliases: ["stopReason error/aborted", errorMessage, errors-as-stream-terminal-events, provider-error-normalization, stable-error-categories, provider-failure-diagnostics, scratch-field-stripping, normalizeProviderError, formatProviderError, lazyStream, MessageV2.fromError, parseStreamError, ResponseStreamError, ContentFilterError]
harnesses: [pi, opencode]
---
Once a model stream has been handed back, failures never throw. The stream ends with a terminal error event whose message keeps the partial content, partial usage, a stable error text and structured diagnostics, and contains no streaming scratch state.

## Why
- When a stream can throw or end with an error, every consumer needs two error paths. Partial output and usage are lost, and aborted turns show zero tokens ([[streamed-usage-misread]]).
- Retry and overflow classifiers usually read the error text. If adapters format errors inconsistently, transient failures are not retried ([[error-text-breaks-retry-classification]], [[foreign-sdk-error-shape-skips-retry]]) or throttling looks like overflow ([[rate-limit-misread-as-overflow]]).
- Errors from proxies and gateways often lose the HTTP status and body ([[provider-error-body-hidden]]).
- Partial messages get persisted. Stream-only buffers then leak into saved tool calls and corrupt resumed sessions ([[stream-scratch-state-persisted]]).
- Billed calls whose response parsing fails lose their cost record ([[billed-call-lost-on-parse-error]]).

## Design space
- **Error transport**
  - Throw or reject.
  - Errors are results: the stream ends with `stopReason: error|aborted` plus `errorMessage`. *pi chose this.*
  - Setup failures (auth, lazy import) may emit an error with no `start` event.
- **Abort vs error**
  - One shared reason.
  - Split into `aborted` and `error`, with the aborted state taken from the signal rather than from error text. *pi chose this:* 2296dc405.
- **Error classification input**
  - Regex over the free-text message. *This is pi's current approach,* and it forces adapters to keep their message wording stable, e.g. Bedrock's fixed prefixes and Mistral's "(server error)" suffix.
  - Structured error kind/code fields. *pi has not done this:* diagnostics exist but classifiers ignore them (unverified plan).
- **Error body enrichment**
  - SDK message only.
  - Normalize status and body across SDK shapes, cap the length, and never replace the original message with a lower-confidence source. *pi chose this.*
- **Usage on failure**
  - Read usage at end of stream only.
  - Capture usage from the first event and patch it with later deltas. *pi chose this.*
  - Set usage before parsing the response, so a malformed but billed response is still accounted.
- **Scratch state**
  - Persist content blocks as they are.
  - Strip `partialJson`, indexes and byte buffers on every terminal path. *pi chose this.*
- **Diagnostics channel**
  - Rely only on `errorMessage` text.
  - Attach a redacted structured `diagnostics[]` array: status, code, request id, transport phase, input transformations.
- **Error storage**: typed error object stored on the assistant message (`AbortedError`, `APIError`, `ContextOverflowError`, `ContentFilterError`…) (opencode legacy) · typed `reason` classes decided at the HTTP boundary (opencode v2).
- **Content-filter finish**: surfaced as a visible error rather than a silent empty stop (opencode `e2527db3c7`).

## Implementations
- [[pi--errors-as-stream-events|pi]] — the StreamFunction contract plus `lazyStream`. Adapter catch blocks strip scratch fields and set aborted/error with the partial message. Errors go through `normalizeProviderError`/`formatProviderError` (4000-char body cap) and diagnostics records.
- [[opencode--errors-as-stream-events|opencode]] — `MessageV2.fromError` maps SDK/fetch errors to typed session errors; `parseStreamError` reads mid-stream codes; v2 typed reasons.

## Failures
- [[provider-error-body-hidden]]
- [[error-text-breaks-retry-classification]]
- [[foreign-sdk-error-shape-skips-retry]]
- [[stream-scratch-state-persisted]]
- [[streamed-usage-misread]]
- [[truncated-stream-accepted-as-success]]
- [[retry-classifier-regex-sprawl]]

## Related
[[unified-provider-api]] · [[auto-retry-backoff]] · [[terminal-event-required]] · [[context-overflow-detection]] · [[usage-cost-accounting]] · [[partial-message-persistence]] · [[abort-propagation]] · [[transcript-replay-repair]]
