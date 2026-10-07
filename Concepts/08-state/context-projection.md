---
type: concept
stage: state
tier: candidate
aliases: [buildSessionProjection, buildContextEntries, prepareRequest projection, head marker, canonical-context-projection, head-marker-context, "context({at})", filterCompacted, MessageV2.latest, SessionHistory, entriesForRunner, Session History]
harnesses: [pi, opencode]
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
- **Reordering projection**: stream newest-first to the latest completed compaction (or its `tail_start_id`), then reorder to `[compaction, summary, retained tail, newer]` — array order stops being chronological, so "latest" must come from timestamps (opencode legacy) → [[reordered-context-misidentifies-latest-turn]].
- **SQL projection by event sequence** with two cutoffs in one query: `seq ≥ latest compaction`, plus `system` updates only after the epoch's `baseline_seq` (opencode v2).

## Implementations
- [[pi--context-projection|pi]] — `prepareRequest` swaps `context.messages` for `sessionManager.buildSessionProjection().messages`; walk leaf→root, latest compaction, `context_edit` overlay; durable `harness/context.ts` derives from head markers with retained-range cache.
- [[opencode--context-projection|opencode]] — legacy `filterCompacted` over SQLite pages re-read every step, tail reorder, `latest()` by time then id; v2 `SessionHistory.entriesForRunner` by `seq` with compaction and Context Epoch cutoffs.

## Failures
- [[timestamp-ordered-history-pagination]]
- [[abandoned-attempts-left-in-context]]
- [[per-request-projection-rescans-log]]
- [[quadratic-long-session-operations]]
- [[reordered-context-misidentifies-latest-turn]]

## Tradeoffs
- [[session-store-format]]

## Related
[[session-tree]] · [[context-edit-overlay]] · [[auto-compaction]] · [[compaction-cut-point]] · [[context-transform-hook]] · [[message-conversion-layer]] · [[transcript-replay-repair]] · [[transcript-carried-system-prompt]] · [[token-estimation]] · [[turn-lifecycle-hooks]] · [[durable-execution]]
