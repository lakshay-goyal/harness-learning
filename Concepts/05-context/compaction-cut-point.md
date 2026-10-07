---
type: concept
stage: compaction
tier: candidate
aliases: [findCutPoint, findProjectedCutPoint, isCutPointMessage, isTurnStartMessage, selectCut, firstKeptEntryId, "keep-recent window", COMPACT_USER_MESSAGE_MAX_TOKENS, RETAINED_MESSAGE_TOKEN_BUDGET, build_v2_compacted_history, retained-messages]
harnesses: [pi, codex]
---
Rules choosing where history splits into "summarize" vs "keep verbatim": walk back from newest until the keep budget is reached, then snap to a legal boundary that never separates a tool call from its result.

## Why
- Providers reject a tool result without its preceding call (and many reject a call without its result); a cut between them makes every later request invalid.
- If the tail alone (e.g. one giant tool result) exceeds the keep budget and the fallback keeps *everything*, compaction silently does nothing and the next request overflows (pi: [[oversized-trailing-tool-results-uncompactable]]).
- The budget estimator must count every message kind the context contains, otherwise the kept window is bigger than intended (pi: custom messages uncounted, [[estimator-undercounts-context]]).

## Design space
- **Budget walk**: tokens from newest backwards until ≥ keepRecent ✔ pi (20000) vs fixed message count vs **recent *user* messages only, newest-first up to a token budget; the boundary message truncated to the remainder** ✔ codex (local 20,000; remote 64,000).
- **Filter instead of cut** ✔ codex: no position-based cut at all — assistant messages, reasoning, tool calls and outputs are dropped wholesale, so the call/result adjacency problem disappears; kept = real user messages (+ hook prompts, selected inter-agent messages, optionally client developer messages on remote).
- **Modality in the keep budget**: text only, media dropped from the truncated boundary message ✔ codex local; images charged at their estimate and kept atomic with their label tags, no older backfill after a boundary image doesn't fit ✔ codex remote (`CompactionImageBudget`).
- **Exclude prior summaries / huge or progress-only agent messages from the kept set** ✔ codex.
- **Legal cut entries**: user/assistant/custom/summary messages, never tool results ✔ pi; durable variant additionally rejects a user entry while a result of the preceding assistant's calls still follows it (late/interleaved results).
- **Snap direction**: first legal point at/after the budget boundary ✔ pi (keeps ≤ budget… plus the turn) vs before (keeps ≥ budget).
- **Fallback when no legal point after boundary**: keep everything (pi before `8bdcd4498`) vs **last legal point** ✔ pi now.
- **Cut mid-turn allowed?** yes → [[split-turn-summary]] ✔ pi; alternative: only cut at turn starts (may keep far more than budget).
- **Metadata adjacency**: pull cut back over context-invisible entries (model change, labels) ✔ pi.
- **Abandoned retry attempts**: advance cut past a suffix that is only an omitted failed attempt so recovery can summarize it ✔ pi (context-edit aware).
- Operate on raw log vs on the **projected context** (after edits/omissions) ✔ pi (`findProjectedCutPoint`; raw `findCutPoint` still exported).

## Implementations
- [[pi--compaction-cut-point|pi]] — backward token walk over projected entries, legal cut = non-toolResult message entry, fallback to last cut point, split-turn detection, recovery-omission suffix advance; durable `selectCut` with interleaved-result rule.
- [[codex--compaction-cut-point|codex]] — keep-set filter, not a cut: newest-first user messages within 20k (local) / 64k (remote) tokens; everything assistant/tool dropped.

## Failures
- [[oversized-trailing-tool-results-uncompactable]]
- [[estimator-undercounts-context]]
- [[repeated-compaction-drops-kept-messages]]
- [[compaction-loses-modality-or-structure]]
- [[summarization-request-overflows]]

## Related
[[auto-compaction]] · [[split-turn-summary]] · [[token-estimation]] · [[context-projection]] · [[context-edit-overlay]] · [[transcript-replay-repair]] · [[overflow-recovery]] · [[compaction-locus]]
