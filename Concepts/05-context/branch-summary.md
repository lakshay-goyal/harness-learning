---
type: concept
stage: compaction
tier: candidate
aliases: [branch_summary, generateBranchSummary, navigateTree, BRANCH_SUMMARY_PROMPT, BRANCH_SUMMARY_PREAMBLE, BRANCH_SUMMARY_PREFIX]
harnesses: [pi]
---
When the user jumps back to an earlier point of a tree-shaped session, optionally summarize the abandoned branch (from old leaf to the common ancestor) and attach the summary at the destination so the new branch knows what was tried.

## Why
- Tree navigation discards context the user may still care about ("tried approach A, it failed because…"); re-explaining is costly, but replaying the whole branch defeats the point of branching.
- Common-ancestor and source-leaf bookkeeping is easy to get wrong (pi: [[branch-summary-wrong-common-ancestor]], [[branch-summary-records-wrong-source-leaf]]).
- Output cap too small → summaries truncated, especially when reasoning consumes the cap ([[summary-output-budget-misfit]]).

## Design space
- Ask each navigation (No summary / Summarize / Summarize with custom prompt) ✔ pi vs always/never; `skipPrompt` setting.
- Range: old leaf → deepest common ancestor, ignoring compaction boundaries (compaction summaries become input) ✔ pi.
- Budget: newest-first fill up to `window − reserve`, summary entries rescued if <90% budget used, tool results skipped (context lives in the call) ✔ pi.
- Prompt: compaction template minus Critical Context ✔ pi; user may *replace* the whole prompt (`replaceInstructions`) — compaction has no equivalent.
- Placement: summary entry becomes child of the target; if target is a user message, leaf = its parent and the text returns to the editor ✔ pi.
- Framing: stored summary already starts with a preamble, render adds a second prefix (pi double-frames — quirk).
- Concurrency: refuse while streaming or compacting ✔ pi (`e687434a6`).
- codex: absent — linear rollout, no tree navigation; `thread/revert` writes a new rollout file with the prefix before a turn and no summary of the dropped suffix (`4ef836f883`, `4343b2bdc4`); fork cuts before the nth user message (`codex-rs/core/src/thread_rollout_truncation.rs:35-94`) → [[session-fork]].

## Implementations
- [[pi--branch-summary|pi]] — `/tree` → `navigateTree` → `collectEntriesForBranchSummary` + `prepareBranchEntries` + `generateBranchSummary` (4096-token cap) → `branch_summary` entry with `fromId`.

## Failures
- [[branch-summary-wrong-common-ancestor]]
- [[branch-summary-records-wrong-source-leaf]]
- [[summary-output-budget-misfit]]
- Cross-group: [[compaction-cancellation-races]] (tree navigation during compaction), [[side-phase-input-lost]] (input during branch summarization)
- [[summary-call-not-retried]] (05-context) — A transient stream drop (terminated, socket close) during the summarization call failed the whole compaction…

## Related
[[session-tree]] · [[session-fork]] · [[structured-compaction-summary]] · [[file-op-tracking]] · [[summary-validation]] · [[auto-compaction]] · [[session-handoff]]
