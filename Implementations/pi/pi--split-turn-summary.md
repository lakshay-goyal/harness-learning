---
type: implementation
harness: pi
concept: split-turn-summary
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/compaction.ts:863, packages/coding-agent/src/core/compaction/compaction.ts:942, packages/coding-agent/src/core/compaction/compaction.ts:994, packages/coding-agent/src/core/compaction/compaction.ts:1077]
---
[[split-turn-summary]] in [[pi]].

## Mechanism
- Detection in cut-point search: cut entry not a turn start + a turn start exists earlier → `isSplitTurn` (`packages/coding-agent/src/core/compaction/compaction.ts:863-869`); `turnPrefixMessages` = projected `[turnStartIndex, firstKeptEntryIndex)` minus compaction entries and system messages (`packages/coding-agent/src/core/compaction/compaction.ts:908-912`); history range ends at `turnStartIndex` (`packages/coding-agent/src/core/compaction/compaction.ts:903`).
- `compact()` split branch (`packages/coding-agent/src/core/compaction/compaction.ts:994-1033`):
  1. `historyText = previousSummary ?? "No prior history."`; only if `messagesToSummarize.length > 0` call `generateSummaryWithUsage` (initial or update prompt, with `customInstructions`).
  2. **then** (sequentially) `generateTurnPrefixSummary`.
  3. Merge: `${historyText}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefix}`; usage combined via `combineUsage`.
- `generateTurnPrefixSummary` (`packages/coding-agent/src/core/compaction/compaction.ts:1076-1117`): `maxTokens = min(floor(0.5·reserve), model.maxTokens)`; `convertToLlm` → `serializeConversation`; prompt text = `# Conversation\n{text}\n\n# Instructions\n{TURN_PREFIX_SUMMARIZATION_PROMPT}` (headings, not XML, `packages/coding-agent/src/core/compaction/compaction.ts:1096`); same system prompt, `completeSummarization` (no cache, retry); rejects error/length ("Turn prefix summarization failed…") and tool calls ("Turn prefix summarization attempted to call a tool").
- File ops also extracted from turn-prefix messages (`packages/coding-agent/src/core/compaction/compaction.ts:917-924`).

### `TURN_PREFIX_SUMMARIZATION_PROMPT` at HEAD (`packages/coding-agent/src/core/compaction/compaction.ts:942-955`, verbatim)
```
The messages above are earlier context from an ongoing conversation. Later messages are stored separately and do not need to be reconstructed.

Create a concise checkpoint of the user's request and the progress shown above. This checkpoint will be placed before the later messages so the conversation can continue with the necessary context.

## Original Request
[What did the user ask for?]

## Progress So Far
- [Key decisions and work completed in these messages]

## Context Needed to Continue
- [Information from these messages needed to understand the later work]

Only summarize information explicitly present above. Do not infer or recreate later messages.
```

### Prompt timeline
| date | hash | wording |
|---|---|---|
| 2025-12-09 | `a38e61909` | v1 "You are performing a CONTEXT CHECKPOINT COMPACTION for a split turn. This is the PREFIX of a turn that was too large to keep in full. The SUFFIX (recent work) is being kept…" |
| 2025-12-09 | `5a9d844f9` | "Merge turn prefix summary into main summary" (single call) |
| 2025-12-29 | `ac71aac09` | structured: `## Original Request / ## Early Progress / ## Context for Suffix` |
| 2026-01-14 | `72e29bce2` (#738) | prefix also serialized as text + summarization system prompt |
| 2026-07-01 | `f58c11562` (#5536) | history + prefix calls changed from `Promise.all` to sequential (single-concurrency local providers returned 429 to overlapping summary generations, per 06-fixes `parallel-side-requests-single-slot-provider`) |
| 2026-09-22 | `d192bd6dc` (#9908 fixes #9652) | rewritten to avoid Claude Fable 5.1 refusals: "Separate conversation content from instructions and replace prefix/suffix wording with continuation-oriented summarization guidance. Hopefully this avoids the refusals." `<conversation>` → `# Conversation`/`# Instructions`; added "Only summarize information explicitly present above. Do not infer or recreate later messages." |

## Constants
| name | value | path:line |
|---|---|---|
| turn-prefix `maxTokens` | `min(floor(0.5·reserveTokens), model.maxTokens)` = 8192 | `packages/coding-agent/src/core/compaction/compaction.ts:1090-1093` |
| separator | `\n\n---\n\n**Turn Context (split turn):**\n\n` | `packages/coding-agent/src/core/compaction/compaction.ts:1032` |

## Evolution
See prompt timeline above; `a38e61909` → `5a9d844f9` → `ac71aac09` → `72e29bce2` → `f58c11562` → `d192bd6dc`.

## Evidence commits
`a38e61909` `5a9d844f9` `ac71aac09` `72e29bce2` `f58c11562` `d192bd6dc`

## Quirks
- Turn-prefix call ignores `customInstructions` (`/compact` focus only reaches the history summary) (`packages/coding-agent/src/core/compaction/compaction.ts:1076-1117`, signature has no instructions param).
- Main summary still uses `<conversation>` XML tags although the prefix prompt moved to headings after the refusal fix — open whether main prompt will follow.
- If `messagesToSummarize` is empty but a previous summary exists, the previous summary is copied verbatim into the new checkpoint (no update pass).
- Durable package has **no** split-turn path: cut at any user/assistant entry, prefix summarized as part of the range.

## Failures
[[summarizer-refusal]] · [[parallel-side-requests-single-slot-provider]]
