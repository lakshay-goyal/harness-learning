---
type: implementation
harness: pi
concept: compaction-cut-point
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/compaction.ts:351, packages/coding-agent/src/core/compaction/compaction.ts:366, packages/coding-agent/src/core/compaction/compaction.ts:446, packages/coding-agent/src/core/compaction/compaction.ts:802, packages/coding-agent/src/core/compaction/compaction.ts:872, packages/durable/src/harness/compaction.ts:255]
---
[[compaction-cut-point]] in [[pi]].

## Mechanism
- **Two implementations**: legacy `findCutPoint` over raw `SessionEntry[]` (`packages/coding-agent/src/core/compaction/compaction.ts:446-501`, still exported) and live `findProjectedCutPoint` over projected entries (`packages/coding-agent/src/core/compaction/compaction.ts:802-870`), used by `prepareCompaction` (`packages/coding-agent/src/core/compaction/compaction.ts:898`). Projected = after `context_edit` omissions/replacements ([[pi--context-projection]]).
- **Legal cut messages** `isCutPointMessage` (`packages/coding-agent/src/core/compaction/compaction.ts:351-364`): `user`, `assistant`, `bashExecution`, `custom`, `branchSummary`, `compactionSummary`; **never `toolResult`** ("they must follow their tool call", `packages/coding-agent/src/core/compaction/compaction.ts:388-393`); system role not a cut point. Compaction entries skipped as cut points (`packages/coding-agent/src/core/compaction/compaction.ts:398-400`, `packages/coding-agent/src/core/compaction/compaction.ts:811`). Cutting at an assistant with tool calls keeps its results (they follow).
- **Turn start** `isTurnStartMessage` (`packages/coding-agent/src/core/compaction/compaction.ts:366-379`): user, bashExecution, custom, branchSummary, compactionSummary (not assistant/toolResult).
- **Algorithm** (`packages/coding-agent/src/core/compaction/compaction.ts:817-829`): collect cut points in `[start,end)`; none → `firstKeptEntryIndex = startIndex` (keep all, `packages/coding-agent/src/core/compaction/compaction.ts:813-815`). Walk backwards summing `estimateTokens` of each projected entry's messages (zero-token entries skipped); once `accumulated ≥ keepRecentTokens`, `cutIndex = cutPoints.find(c => c >= i) ?? cutPoints.at(-1)` (first legal point at/after the boundary, else the **last** one — fix `8bdcd4498`). Budget never exceeded → `cutIndex = cutPoints[0]` (keep everything from first legal point).
- **Recovery-omission suffix advance** (`packages/coding-agent/src/core/compaction/compaction.ts:831-856`): if budget exceeded and the suffix after the cut consists only of context-invisible entries, contains an omitted (edited-away) assistant attempt, and no "external replacement" edit → `cutIndex++`. Rationale: let "an over-budget recovered input be summarized while retaining the edits that keep the abandoned attempt omitted" (`packages/coding-agent/docs/compaction.md:138`).
- **Metadata pull-back** (`packages/coding-agent/src/core/compaction/compaction.ts:858-862`): move the cut backwards over adjacent context-invisible entries (model_change, label, …) but not past a compaction.
- **Split detection** (`packages/coding-agent/src/core/compaction/compaction.ts:863-869`): if the cut entry is not a turn start, search backwards for the turn start; `isSplitTurn = !startsTurn && turnStartIndex !== -1` → [[pi--split-turn-summary]].
- **Range bounds** in `prepareCompaction` (`packages/coding-agent/src/core/compaction/compaction.ts:872-936`): returns `undefined` if last path entry is already a compaction (`packages/coding-agent/src/core/compaction/compaction.ts:876-878`); `boundaryStart = prevCompactionIndex + 1` in the *projection* (= previous compaction's kept entries, see [[pi--iterative-summary-update]]); `historyEnd = isSplitTurn ? turnStartIndex : firstKeptEntryIndex` (`packages/coding-agent/src/core/compaction/compaction.ts:903`); `firstKeptEntry` must have an id; nothing to summarize → `undefined` (`packages/coding-agent/src/core/compaction/compaction.ts:914`).
- Output shape `CompactionPreparation{firstKeptEntryId, messagesToSummarize, turnPrefixMessages, isSplitTurn, tokensBefore, previousSummary, fileOps, settings}` exposed to extensions (`packages/coding-agent/src/core/compaction/compaction.ts:772-788`).

## Constants
| name | value | path:line |
|---|---|---|
| `keepRecentTokens` | 20000 | `packages/coding-agent/src/core/compaction/compaction.ts:129` |
| per-message estimate | chars/4, image 4800 chars | `packages/coding-agent/src/core/compaction/compaction.ts:276-349` |

## Evolution
- 2025-12-09 `a38e61909` cut points + split turns introduced.
- 2025-12-31 `ddda8b124` operate on current branch path only.
- 2026-03-27 `eeace7971` (#2608) range starts at previous kept entries.
- 2026-07-09 `a6f720e6c` (#6326) custom messages counted toward keep budget ("Estimate via the same projection that builds context").
- 2026-09-18 `8bdcd4498` (#9740) fallback to last cut point when trailing tool results alone exceed budget.
- 2026-09-21 `466db0fec` projected cut point + recovery-omission suffix advance.

## Evidence commits
`a38e61909` `ddda8b124` `eeace7971` `a6f720e6c` `8bdcd4498` `466db0fec`

## Quirks
- Legacy raw `findCutPoint` remains exported but unused by `prepareCompaction` — kept for tests/extensions? (unverified).
- Cut snaps *forward* to the first legal point ≥ boundary, so the kept window may be smaller than `keepRecentTokens` when the boundary falls inside a long tool sequence (inferred).

## Durable variant (packages/durable)
- `selectCut(view, keepRecentTokens)` (`packages/durable/src/harness/compaction.ts:255-273`): candidates start at index 1 when a head marker exists; candidate = entry whose contribution starts with assistant or user; a **user entry is rejected if a toolResult answering the preceding assistant's calls still follows it** before the next assistant (`isCandidate`, `:275-296`, handles interleaved/late results); same backward walk with `?? candidates.at(-1)` fallback; returns `undefined` unless at least one non-empty contribution precedes the cut (`:271`). Summarized range = contributions before cut with tool results reordered after their calls (`summarizedMessages`, `:298`).

## Failures
[[oversized-trailing-tool-results-uncompactable]] · [[estimator-undercounts-context]] · [[repeated-compaction-drops-kept-messages]]
