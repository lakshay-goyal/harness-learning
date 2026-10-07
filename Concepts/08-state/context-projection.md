---
type: concept
stage: state
tier: candidate
aliases: [buildSessionProjection, buildContextEntries, prepareRequest projection, head marker, canonical-context-projection, head-marker-context, "context({at})", reconstruct_rollout, reconstruct_history_from_rollout, TurnReferenceContextItem]
harnesses: [pi, codex]
---
The model's request context is *derived* from the persistent log on every request (latest compaction/head + kept entries + post-head entries + edit overlay), not maintained as a separately mutated in-memory array — the log is the source of truth for what the model sees.

## Why
- Two sources of truth (in-memory agent messages vs persisted log) drift: retries, compaction, extension edits and branch moves must otherwise be applied twice and consistently; abandoned retry attempts leaked into later requests until projection became authoritative ([[abandoned-attempts-left-in-context]]).
- Lets raw history stay append-only while the model sees a cleaned view (failed attempts omitted, compacted prefix replaced by summary).
- Cost: recomputing per request must be O(n) or cached ([[per-request-projection-rescans-log]], [[quadratic-long-session-operations]]).

## Design space
- **Mutable in-memory context** (pi agent-core default, `Agent.state.messages`) vs **projection from log per request** (pi coding-agent since `466db0fec`; pi-durable always).
- **Boundary marker**: latest compaction entry + `firstKeptEntryId` (pi) vs generic `head` markers (reset or compaction) in durable.
- **Overlay**: append-only edit entries (omit/replace) applied in projection ([[context-edit-overlay]]).
- **Normalization at projection**: tool results moved after their call, missing results synthesized, failed/aborted/deferred assistants excluded, leading system message (durable `context.ts`) vs at provider boundary ([[transcript-replay-repair]], pi coding-agent).
- **System prompt/tools in projection**: replayed `SystemMessage` deltas and compaction-time checkpoint ([[transcript-carried-system-prompt]]).
- **Caching**: rescan per request vs per-invocation range reuse vs in-memory last range with retention TTL (durable).
- **Projection only at resume/fork, mutable history in between** ✔ codex (reverse scan to newest surviving `replacement_history` compaction, forward replay suffix).
- **Re-apply current model's truncation on replay** ✔ codex.
- **Context baseline diff** (`reference_context_item`): inject only changed context per turn; missing baseline ⇒ full re-injection ✔ codex → [[world-state-diff-injection]].

## Implementations
- [[pi--context-projection|pi]] — `prepareRequest` swaps `context.messages` for `sessionManager.buildSessionProjection().messages`; walk leaf→root, latest compaction, `context_edit` overlay; durable `harness/context.ts` derives from head markers with retained-range cache.
- [[codex--context-projection|codex]] — projection only at resume/fork: reverse scan to newest surviving compaction with replacement history, forward replay of suffix under current model's truncation; live turns use a mutable in-memory ContextManager diffed against a context baseline.

## Failures
- [[abandoned-attempts-left-in-context]]
- [[per-request-projection-rescans-log]]
- [[quadratic-long-session-operations]]
- [[compaction-drops-harness-state]]

## Related
[[session-tree]] · [[context-edit-overlay]] · [[auto-compaction]] · [[compaction-cut-point]] · [[context-transform-hook]] · [[message-conversion-layer]] · [[transcript-replay-repair]] · [[transcript-carried-system-prompt]] · [[token-estimation]] · [[turn-lifecycle-hooks]] · [[durable-execution]] · [[session-log-shape]]
