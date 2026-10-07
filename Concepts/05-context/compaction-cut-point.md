---
type: concept
stage: compaction
tier: candidate
aliases: [findCutPoint, findProjectedCutPoint, isCutPointMessage, isTurnStartMessage, selectCut, firstKeptEntryId, "keep-recent window", tail_start_id, preserve_recent_tokens, tail_turns, splitTurn, MIN_PRESERVE_RECENT_TOKENS, DEFAULT_KEEP_TOKENS]
harnesses: [pi, opencode]
---
Rules choosing where history splits into "summarize" vs "keep verbatim": walk back from newest until the keep budget is reached, then snap to a legal boundary that never separates a tool call from its result.

## Why
- Providers reject a tool result without its preceding call (and many reject a call without its result); a cut between them makes every later request invalid.
- If the tail alone (e.g. one giant tool result) exceeds the keep budget and the fallback keeps *everything*, compaction silently does nothing and the next request overflows (pi: [[oversized-trailing-tool-results-uncompactable]]).
- The budget estimator must count every message kind the context contains, otherwise the kept window is bigger than intended (pi: custom messages uncounted, [[estimator-undercounts-context]]).

## Design space
- **Budget walk**: tokens from newest backwards until ≥ keepRecent ✔ pi (20000) vs fixed message count vs "last N user turns" (Codex keeps recent *user* messages).
- **Legal cut entries**: user/assistant/custom/summary messages, never tool results ✔ pi; durable variant additionally rejects a user entry while a result of the preceding assistant's calls still follows it (late/interleaved results).
- **Snap direction**: first legal point at/after the budget boundary ✔ pi (keeps ≤ budget… plus the turn) vs before (keeps ≥ budget).
- **Fallback when no legal point after boundary**: keep everything (pi before `8bdcd4498`) vs **last legal point** ✔ pi now.
- **Cut mid-turn allowed?** yes → [[split-turn-summary]] ✔ pi; alternative: only cut at turn starts (may keep far more than budget).
- **Metadata adjacency**: pull cut back over context-invisible entries (model change, labels) ✔ pi.
- **Abandoned retry attempts**: advance cut past a suffix that is only an omitted failed attempt so recovery can summarize it ✔ pi (context-edit aware).
- Operate on raw log vs on the **projected context** (after edits/omissions) ✔ pi (`findProjectedCutPoint`; raw `findCutPoint` still exported).
- **Budget scaled to the window**: `clamp(25% of usable, 2k, 15k)` unless configured (opencode legacy) vs fixed (pi 20000, opencode v2 8000).
- **Turn cap**: optional `tail_turns` limits kept user turns, 0 disables the tail (opencode legacy; default 2 removed `dab2637217`).
- **Split prefix folded into the head summary** rather than summarized separately (opencode legacy `splitTurn`).
- **Kept tail as serialized text, not provider messages**: legality only matters at line level and no signed/encrypted reasoning crosses the boundary (opencode v2 `<recent-context>`).
- **Boundary persisted by id** on the compaction marker (`tail_start_id`, opencode legacy) — must be remapped on fork ([[fork-boundary-loss]]).

## Implementations
- [[pi--compaction-cut-point|pi]] — backward token walk over projected entries, legal cut = non-toolResult message entry, fallback to last cut point, split-turn detection, recovery-omission suffix advance; durable `selectCut` with interleaved-result rule.
- [[opencode--compaction-cut-point|opencode]] — legacy: newest-first whole user turns within a window-scaled budget, first non-fitting turn split by suffix, tail start persisted as `tail_start_id`; v2: newest serialized lines within 8k tokens kept as `recent` text.

## Failures
- [[oversized-trailing-tool-results-uncompactable]]
- [[estimator-undercounts-context]]
- [[repeated-compaction-drops-kept-messages]]
- [[reordered-context-misidentifies-latest-turn]] (08-state)
- Cross-group: [[fork-boundary-loss]] (08-state)

## Tradeoffs
- [[compaction-design]]

## Related
[[auto-compaction]] · [[split-turn-summary]] · [[token-estimation]] · [[context-projection]] · [[context-edit-overlay]] · [[transcript-replay-repair]] · [[overflow-recovery]]
