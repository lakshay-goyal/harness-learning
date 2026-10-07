---
type: concept
stage: failure-handling
tier: candidate
aliases: [_prepareRetry, settings.retry, isRetryableAssistantError, RETRYABLE_PROVIDER_ERROR_PATTERN, retryProviderRequest, auto_retry_start, auto_retry_end, maxAgentDelayMs, maxRetryDelayMs, two-layer-retry, transient-error-classification]
harnesses: [pi]
---
Harness-level retry of transient provider failures with capped exponential backoff; a classifier decides retryable vs terminal (quota, overflow, deterministic errors).

## Why
- Transient 429/5xx/network/early-EOF failures otherwise end headless runs ("waiting for a manual nudge") ([[retry-classifier-regex-sprawl]]).
- Hidden SDK retries under the agent loop double-retry, burn quota, and ignore cancellation ([[hidden-sdk-retries-double-retry]], [[retry-backoff-hygiene]]).
- Retry budgets scoped wrong exhaust across a tool-use run ([[retry-counter-accumulates-across-turn]]); deterministic failures retried pointlessly ([[deterministic-5xx-retried]]).
- Failed attempts must leave the model's context ([[abandoned-attempts-left-in-context]]); callers need to know a retry is pending ([[retry-wait-race-prompt-returns-early]]).

## Design space
- **Layering**: SDK retries (hidden) · harness-owned single visible layer (pi after `8fb1e877c`) · two layers with escalation: adapter retries only pre-headers and throws long server delays to the visible layer (pi).
- **Classifier**: regex over error text (pi; grows forever, couples adapters to wording) · structured error kinds/status codes · header hints (`x-should-retry`, `Retry-After`).
- **Backoff**: `base·2^(n−1)` capped, no jitter (pi agent) · capped exponential with jitter (pi adapter) · server-specified delay with validation.
- **Budget scope**: per request, reset on success (pi `4f004adef`) · per user turn.
- **Exclusions**: context overflow → compaction instead (pi) · quota/billing → terminal (pi).
- **Context hygiene**: delete failed attempt · hide via append-only edit (pi `context_edit`) · keep.
- **Durability**: in-memory sleep (pi stable) · checkpointed retry phase with absolute deadline (pi-durable).

## Implementations
- [[pi--auto-retry-backoff|pi]] — `settings.retry` 3 × 2s/4s/8s capped 60 s; regex classifier in pi-ai; abortable adapter retry with 60 s server-delay escalation; failed attempts omitted via `context_edit`.

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

## Related
[[terminal-event-required]] · [[errors-as-stream-events]] · [[context-overflow-detection]] · [[overflow-recovery]] · [[context-edit-overlay]] · [[run-settlement]] · [[abort-propagation]] · [[http-transport-hardening]] · [[virtual-model-router]]
