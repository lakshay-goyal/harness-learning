---
type: implementation
harness: pi
concept: branch-summary
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/branch-summarization.ts:108, packages/coding-agent/src/core/compaction/branch-summarization.ts:195, packages/coding-agent/src/core/compaction/branch-summarization.ts:253, packages/coding-agent/src/core/compaction/branch-summarization.ts:293, packages/coding-agent/src/core/agent-session.ts:3963, packages/coding-agent/src/core/session-manager.ts:1600, packages/coding-agent/src/core/messages.ts:19, packages/coding-agent/src/modes/interactive/interactive-mode.ts:5568]
---
[[branch-summary]] in [[pi]].

## Mechanism
- **Trigger**: `/tree` navigation; TUI asks "Summarize branch?" — No summary / Summarize / Summarize with custom prompt (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:5568-5591`); `branchSummary.skipPrompt` (default false) skips the question and defaults to no summary (`packages/coding-agent/src/core/settings-manager.ts:35-38`, `999-1008`).
- `navigateTree(targetId, {summarize, customInstructions, replaceInstructions, label})` (`packages/coding-agent/src/core/agent-session.ts:3963-4155`): refuses while streaming or compacting (`packages/coding-agent/src/core/agent-session.ts:3967-3974`, `e687434a6`); `session_before_tree` hook can cancel, supply a summary, override `customInstructions` / `replaceInstructions` / `label` (`packages/coding-agent/src/core/extensions/types.ts:1486-1499`, `packages/coding-agent/src/core/agent-session.ts:4023-4050`); default summarizer otherwise; `session_tree` after.
- **Range** `collectEntriesForBranchSummary` (`packages/coding-agent/src/core/compaction/branch-summarization.ts:108-146`): deepest common ancestor = iterate the root-first target path *backwards* and take the first id on the old path (`92947a3dc`); collect old leaf → ancestor, reverse. Does **not** stop at compaction boundaries; compaction and branch summaries become input messages (`packages/coding-agent/src/core/compaction/branch-summarization.ts:99-101`, `packages/coding-agent/src/core/compaction/branch-summarization.ts:169-170`).
- **Budget** `prepareBranchEntries(entries, tokenBudget)` (`packages/coding-agent/src/core/compaction/branch-summarization.ts:195-247`): first pass gathers file ops from all nested pi-generated branch summaries; second pass newest→oldest adding messages until `tokenBudget`; tool results skipped ("context is in assistant's tool call", `packages/coding-agent/src/core/compaction/branch-summarization.ts:159-160`); a compaction/branch-summary entry that doesn't fit is still added if `totalTokens < 0.9·budget`, then stop.
- **Generation** `generateBranchSummary` (`packages/coding-agent/src/core/compaction/branch-summarization.ts:293-375`): `tokenBudget = (model.contextWindow || 128000) − reserveTokens(16384)`; nothing → `"No content to summarize"`; `convertToLlm` → `serializeConversation`; instructions = `customInstructions` alone if `replaceInstructions`, else `BRANCH_SUMMARY_PROMPT + "\n\nAdditional focus: …"`; prompt `<conversation>…</conversation>\n\n{instructions}`; `maxTokens = min(4096, model.maxTokens)` (`packages/coding-agent/src/core/compaction/branch-summarization.ts:345`); `completeSummarization` (no cache, fresh uuid session id, retry policy); **no `reasoning` option passed**; abort → `{aborted:true}`; error/length/toolCall → `{error}`; result = `BRANCH_SUMMARY_PREAMBLE + text + file lists`.
- **Placement** `branchWithSummary(branchFromId, summary, details, fromHook, usage)` (`packages/coding-agent/src/core/session-manager.ts:1600-1625`): `fromId = leafId ?? "root"` (old leaf), new `branch_summary` entry child of the target, leaf moves to it. If the target is a user/custom message, leaf = its parent and its text is put back in the editor (`packages/coding-agent/src/core/agent-session.ts:4091-4102`); optional label attached to the summary entry.
- **Rendering**: `branchSummary` role → user message `"The following is a summary of a branch that this conversation came back from:\n\n<summary>\n" + summary + "</summary>"` (`packages/coding-agent/src/core/messages.ts:19-24`, `170-175`).

### `BRANCH_SUMMARY_PREAMBLE` + `BRANCH_SUMMARY_PROMPT` (`packages/coding-agent/src/core/compaction/branch-summarization.ts:253-286`, verbatim)
```
The user explored a different conversation branch before returning here.
Summary of that exploration:

```
```
Create a structured summary of this conversation branch for context when returning later.

Use this EXACT format:

## Goal
[What was the user trying to accomplish in this branch?]

## Constraints & Preferences
- [Any constraints, preferences, or requirements mentioned]
- [Or "(none)" if none were mentioned]

## Progress
### Done
- [x] [Completed tasks/changes]

### In Progress
- [ ] [Work that was started but not finished]

### Blocked
- [Issues preventing progress, if any]

## Key Decisions
- **[Decision]**: [Brief rationale]

## Next Steps
1. [What should happen next to continue this work]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```

## Constants
| name | value | path:line |
|---|---|---|
| output cap | `min(4096, model.maxTokens)` (was 2048) | `packages/coding-agent/src/core/compaction/branch-summarization.ts:345` |
| `branchSummary.reserveTokens` | 16384 | `settings-manager.ts:1001`; `packages/coding-agent/src/core/compaction/branch-summarization.ts:305` |
| fallback window | 128000 | `packages/coding-agent/src/core/compaction/branch-summarization.ts:311-313` |
| summary rescue | < 0.9 × budget | `packages/coding-agent/src/core/compaction/branch-summarization.ts:231-237` |

## Evolution
- 2025-12-29 `ac71aac09` structured format shared with compaction; `8fe8fe992` preamble; `04f2fcf00` XML file lists; `4ef3325ce`/`d1a49c45f` file lists in details/text; `92947a3dc` deepest common ancestor fix; `dc5fc4fc4` `maxTokens = 100000` budget replaced by `window − reserveTokens`.
- 2025-12-31 `d103af4ca` cumulative file tracking.
- 2026-03-04 `5c61d6bc9` (#1803) queue messages during branch summarization; 2026-04-13 `3166dae72` (#3091) flush after tree navigation.
- 2026-07-20 `2fd386840` (#6671) usage stored on branch summary entries; 2026-07-21 `8e53e0e49` retry policy; 2026-07-23 `241431c69` no cache writes.
- 2026-08-19 `d711bd5f0` `fromId` records source leaf (was destination).
- 2026-09-03 `e44d75c20` (#8845) cap 2048 → 4096 (reasoning consumed the cap); `bea67d90d` (#8920) branch summary counted as busy for abort.
- 2026-09-07 `e687434a6` (#9179) reject tree navigation during compaction.

## Evidence commits
`ac71aac09` `8fe8fe992` `04f2fcf00` `4ef3325ce` `d1a49c45f` `92947a3dc` `dc5fc4fc4` `d103af4ca` `5c61d6bc9` `3166dae72` `2fd386840` `8e53e0e49` `241431c69` `d711bd5f0` `e44d75c20` `bea67d90d` `e687434a6`

## Quirks
- **Double framing**: stored summary already starts with `BRANCH_SUMMARY_PREAMBLE`, and rendering adds `BRANCH_SUMMARY_PREFIX` → model sees two preambles (open question whether intentional).
- `BRANCH_SUMMARY_SUFFIX = "</summary>"` lacks the leading newline that `COMPACTION_SUMMARY_SUFFIX` has (`messages.ts:17` vs `:24`) — cosmetic.
- Branch summary never sends `reasoning` even for reasoning models (compare compaction which reuses session thinking level).
- Leaf on load = last file entry: a `/tree` move without summary/label appends nothing and is lost on resume (01-loop finding, inferred).
- Durable experimental TUI lacks tree navigation entirely (`packages/coding-agent/src/experimental/durable/README.md:66`).

## Failures
[[branch-summary-wrong-common-ancestor]] · [[branch-summary-records-wrong-source-leaf]] · [[summary-output-budget-misfit]] · cross-group: [[compaction-cancellation-races]], [[side-phase-input-lost]]
