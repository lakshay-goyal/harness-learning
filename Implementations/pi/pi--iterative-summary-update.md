---
type: implementation
harness: pi
concept: iterative-summary-update
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/compaction.ts:540, packages/coding-agent/src/core/compaction/compaction.ts:577, packages/coding-agent/src/core/compaction/compaction.ts:717, packages/coding-agent/src/core/compaction/compaction.ts:883, packages/coding-agent/src/core/session-manager.ts:558]
---
[[iterative-summary-update]] in [[pi]].

## Mechanism
- `prepareCompaction` (`packages/coding-agent/src/core/compaction/compaction.ts:883-895`): `prevCompactionIndex` = projected compaction entry with non-empty messages (only the newest compaction projects; older ones inside the retained range project `[]`, `packages/coding-agent/src/core/session-manager.ts:558-564`); `previousSummary = prevCompaction.summary`; `boundaryStart = prevCompactionIndex + 1` → in the canonical projection that is the previous compaction's **kept** entries, so messages that survived the last compaction get summarized this time.
- `generateSummaryWithUsage` (`packages/coding-agent/src/core/compaction/compaction.ts:696-768`): `basePrompt = previousSummary ? UPDATE_SUMMARIZATION_PROMPT : SUMMARIZATION_PROMPT`; prompt text = `<conversation>…</conversation>\n\n<previous-summary>\n{prev}\n</previous-summary>\n\n{basePrompt}` (+ "Additional focus").
- Previous summary text includes the previously appended `<read-files>`/`<modified-files>` blocks (stored summary = LLM text + file lists); lists are also re-seeded structurally from `details` → [[pi--file-op-tracking]].
- Split turn with empty history range: previous summary reused verbatim (`packages/coding-agent/src/core/compaction/compaction.ts:995`).

### `UPDATE_SUMMARIZATION_PROMPT` (`packages/coding-agent/src/core/compaction/compaction.ts:540-579`, verbatim)
```
The messages above are NEW conversation messages to incorporate into the existing summary provided in <previous-summary> tags.

Update the existing structured summary with new information. RULES:
- PRESERVE all existing information from the previous summary
- ADD new progress, decisions, and context from the new messages
- UPDATE the Progress section: move items from "In Progress" to "Done" when completed
- UPDATE "Next Steps" based on what was accomplished
- PRESERVE exact file paths, function names, and error messages
- If something is no longer relevant, you may remove it

Use this EXACT format:

## Goal
[Preserve existing goals, add new ones if the task expanded]

## Constraints & Preferences
- [Preserve existing, add new ones discovered]

## Progress
### Done
- [x] [Include previously done items AND newly completed items]

### In Progress
- [ ] [Current work - update based on progress]

### Blocked
- [Current blockers - remove if resolved]

## Key Decisions
- **[Decision]**: [Brief rationale] (preserve all previous, add new)

## Next Steps
1. [Update based on current state]

## Critical Context
- [Preserve important context, add new if needed]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```
(Instructions live in `UPDATE_SUMMARIZATION_INSTRUCTIONS` `packages/coding-agent/src/core/compaction/compaction.ts:540-575`; the prompt wrapper `packages/coding-agent/src/core/compaction/compaction.ts:577-579`.)

## Evolution
- 2025-12-29 `09d6131be` "Add file tracking and iterative summary merging to compaction" — update prompt introduced.
- 2026-03-27 `eeace7971` (#2608): previously `boundaryStart` was the entry after the compaction entry in raw file order; kept messages sit *before* the compaction entry in the append-only file, so they were dropped from the next summary and from context. Fix starts at previous `firstKeptEntryId`, recomputes `tokensBefore` from rebuilt context, excludes compaction entries.
- 2026-09-21 `466db0fec` same logic expressed over the canonical projection (`prevCompactionIndex + 1`).

## Evidence commits
`09d6131be` `eeace7971` `466db0fec` `ed0d6b91b`

## Quirks
- "If something is no longer relevant, you may remove it" licenses drift across many compactions; no test of long chains found (unverified).
- Branch summaries don't use the update prompt; nested branch summaries become ordinary input messages.

## Durable variant (packages/durable)
- No `<previous-summary>`; the head marker (previous summary as wrapped user message) is the first message of the serialized range, and the single prompt says "If the conversation starts with an earlier summary, preserve its information and fold the newer messages into it" (`packages/durable/src/harness/compaction.ts:65`).

## Failures
[[repeated-compaction-drops-kept-messages]]
