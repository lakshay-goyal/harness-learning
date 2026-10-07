---
type: concept
stage: failure-handling
tier: candidate
aliases: [_overflowRecoveryAttempted, overflow-compact-and-retry, "reason: overflow"]
harnesses: [pi]
---
When a request overflows (explicit provider error, silent usage > window, or early length stop), hide the failed attempt, compact, and retry exactly once; a second overflow in the same user turn surfaces an error.

## Why
- Estimates are heuristics; real overflows still happen. Without recovery the run dies with a raw provider error.
- Unbounded compact-and-retry loops forever when compaction can't shrink enough ([[overflow-compaction-cascade]]).
- Retrying a *successful* response that merely reported usage > window re-runs a finished answer ([[completed-response-retried-after-overflow]]).
- After switching to a larger model, an old model's overflow error must not trigger compaction ([[overflow-judged-against-wrong-model]]).
- Misclassifying throttling as overflow causes destructive compaction instead of backoff ([[rate-limit-misread-as-overflow]], 02 group).

## Design space
- **Classifier input**: error regexes + silent overflow + length heuristics → [[context-overflow-detection]].
- **Attempts**: one per user turn (latch reset on user message / clean response) ✔ pi; durable: one compaction per generation (`compacted` checkpoint).
- **Retry vs compact separation**: overflow excluded from transient-retry classifier ("handled by compaction, not retry") ✔ pi.
- **Failed attempt handling**: keep in raw log but omit from projection via `context_edit` ✔ pi; durable appends the overflowed assistant then compacts.
- **Successful-but-overflowing**: compact without retry ✔ pi.
- **Length stop below intended max**: treat as context pressure, one compact-and-retry ✔ pi (`isRecoverableLength`, `32850ef7c`); durable treats length as an answer.
- **Model guard**: judge overflow against the model that produced the message ✔ pi; ignore messages older than latest compaction ✔ pi.
- **If no cut exists** (nothing to summarize): durable falls through to error; coding-agent compaction returns without retry.

## Implementations
- [[pi--overflow-recovery|pi]] — `_checkCompaction` cases 1/2: omit failed attempt via `context_edit`, `_runAutoCompaction("overflow", true)`, `agent.continue()`; latch `_overflowRecoveryAttempted`; durable generation compacts once then re-prepares.

## Failures
- [[overflow-compaction-cascade]]
- [[completed-response-retried-after-overflow]]
- [[overflow-judged-against-wrong-model]]
- [[length-stop-recovery]]
- Cross-group: [[rate-limit-misread-as-overflow]], [[overflow-message-not-recognized]] (02-model-interface)

## Related
[[context-overflow-detection]] · [[auto-compaction]] · [[auto-retry-backoff]] · [[context-edit-overlay]] · [[run-settlement]] · [[max-tokens-context-clamp]] · [[token-estimation]]
