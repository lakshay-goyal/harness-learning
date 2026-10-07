---
type: implementation
harness: pi
concept: file-op-tracking
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/utils.ts:12, packages/coding-agent/src/core/compaction/utils.ts:30, packages/coding-agent/src/core/compaction/utils.ts:67, packages/coding-agent/src/core/compaction/utils.ts:77, packages/coding-agent/src/core/compaction/compaction.ts:51, packages/coding-agent/src/core/compaction/compaction.ts:60, packages/coding-agent/src/core/compaction/branch-summarization.ts:200]
---
[[file-op-tracking]] in [[pi]].

## Mechanism
- `FileOperations = {read, written, edited}` sets (`packages/coding-agent/src/core/compaction/utils.ts:12-24`).
- `extractFileOpsFromMessage` (`packages/coding-agent/src/core/compaction/utils.ts:30-61`): assistant `toolCall` blocks whose name is exactly `read` / `write` / `edit` and whose `args.path` is a string; for `toolResult`, iterates `nestedCalls.calls` ("Calls made from codemode scripts are recorded on the script's result"). Everything else (bash `cat`/`sed -i`, grep, MCP, plugin tools) is invisible.
- `computeFileLists` (`packages/coding-agent/src/core/compaction/utils.ts:67-72`): modified = edited ∪ written; readFiles = read − modified; both sorted.
- `formatFileOperations` (`packages/coding-agent/src/core/compaction/utils.ts:77-87`) appended to the summary text:
```
<read-files>
a.ts
</read-files>

<modified-files>
b.ts
</modified-files>
```
- Compaction: `extractFileOperations` seeds from previous compaction `details` **only if `!fromHook`** (pi-generated), prev `modifiedFiles` re-added as `edited` (`packages/coding-agent/src/core/compaction/compaction.ts:60-87`); then messages to summarize + turn-prefix messages (`packages/coding-agent/src/core/compaction/compaction.ts:917-924`); stored as `CompactionEntry.details = {readFiles, modifiedFiles}` (`CompactionDetails`, `packages/coding-agent/src/core/compaction/compaction.ts:51-55`, `packages/coding-agent/src/core/compaction/compaction.ts:1069`).
- Branch summaries: first pass collects details from *all* nested pi-generated `branch_summary` entries (even outside the token budget), second pass from messages within budget (`packages/coding-agent/src/core/compaction/branch-summarization.ts:200-225`). Doc: compaction carries from previous compaction; branch summarization carries from branch summaries; no cross-feed of compaction details into branch first pass (`packages/coding-agent/docs/compaction.md:200-204`).

## Evolution
- 2025-12-29 `09d6131be` file tracking introduced with iterative merging; `e7bfb5afe` read-only vs modified split; `4ef3325ce` lists stored in `BranchSummaryEntry.details`; `d1a49c45f` lists appended to summary text for LLM + TUI; `04f2fcf00` XML tags.
- 2025-12-31 `d103af4ca` cumulative tracking for both compaction and branch summaries.
- 2026-09-29 `8562bcf66` codemode nested calls counted (via `nestedCalls`).

## Evidence commits
`09d6131be` `e7bfb5afe` `4ef3325ce` `d1a49c45f` `04f2fcf00` `d103af4ca` `8562bcf66`

## Quirks
- Tracking ignores bash-driven file access and differently-named extension tools — no evidence of plans to extend (open question).
- Extension-supplied compactions (`fromHook`) break the cumulative chain: next pi compaction starts lists from scratch.
- Lists appear twice to the iterative summarizer (inside `<previous-summary>` text and re-appended) — harmless duplication (inferred).
- Durable harness has no file-op tracking.

## Failures
(none mined)
