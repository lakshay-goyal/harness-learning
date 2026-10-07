---
type: concept
stage: failure-handling
tier: candidate
aliases: [_overflowRecoveryAttempted, overflow-compact-and-retry, "reason: overflow", set_total_tokens_full, remove_first_item, guardian_budget_compacted]
harnesses: [pi, codex]
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
- **Attempts**: one per user turn (latch reset on user message / clean response) ✔ pi; durable: one compaction per generation (`compacted` checkpoint); **zero in-turn for normal sessions: fail the turn, pin usage to the full window so the *next* turn compacts first** ✔ codex ([[no-in-turn-overflow-retry]]; hardening attempt reverted same day `69f3183a8e`); once per model step for approval-review sessions ✔ codex.
- **Overflow of the compaction request itself**: drop the oldest history item (and its paired call/output) and retry — "Trim from the beginning to preserve cache (prefix-based)" ✔ codex; pi caps serialized tool results instead ([[transcript-serialization-for-summary]]).
- **Prevention over recovery**: proactive 90 % trigger + 95 % effective window + post-turn and model-downshift compaction ✔ codex.
- **Retry vs compact separation**: overflow excluded from transient-retry classifier ("handled by compaction, not retry") ✔ pi.
- **Failed attempt handling**: keep in raw log but omit from projection via `context_edit` ✔ pi; durable appends the overflowed assistant then compacts.
- **Successful-but-overflowing**: compact without retry ✔ pi.
- **Length stop below intended max**: treat as context pressure, one compact-and-retry ✔ pi (`isRecoverableLength`, `32850ef7c`); durable treats length as an answer.
- **Model guard**: judge overflow against the model that produced the message ✔ pi; ignore messages older than latest compaction ✔ pi.
- **If no cut exists** (nothing to summarize): durable falls through to error; coding-agent compaction returns without retry.

## Implementations
- [[pi--overflow-recovery|pi]] — `_checkCompaction` cases 1/2: omit failed attempt via `context_edit`, `_runAutoCompaction("overflow", true)`, `agent.continue()`; latch `_overflowRecoveryAttempted`; durable generation compacts once then re-prepares.
- [[codex--overflow-recovery|codex]] — deferred recovery: error + `set_total_tokens_full` → next turn's pre-sampling compaction; compaction-request overflow trims oldest-first; guardian reviews compact-and-retry once.

## Failures
- [[overflow-compaction-cascade]]
- [[completed-response-retried-after-overflow]]
- [[overflow-judged-against-wrong-model]]
- [[length-stop-recovery]]
- [[summarization-request-overflows]]
- Cross-group: [[compaction-pinned-to-unavailable-model]] (02-model-interface)
- Cross-group: [[rate-limit-misread-as-overflow]], [[overflow-message-not-recognized]] (02-model-interface)

## Related
[[context-overflow-detection]] · [[auto-compaction]] · [[auto-retry-backoff]] · [[context-edit-overlay]] · [[run-settlement]] · [[max-tokens-context-clamp]] · [[token-estimation]] · [[compaction-locus]]
