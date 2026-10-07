---
type: implementation
harness: pi
concept: structured-compaction-summary
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/compaction.ts:507, packages/coding-agent/src/core/compaction/utils.ts:161, packages/coding-agent/src/core/compaction/branch-summarization.ts:258, packages/durable/src/harness/compaction.ts:65]
---
[[structured-compaction-summary]] in [[pi]].

## Mechanism
- System prompt `SUMMARIZATION_SYSTEM_PROMPT` (`packages/coding-agent/src/core/compaction/utils.ts:161-163`) shared by compaction, split-turn, branch summary (and copied in durable).
- User message = `<conversation>\n{serialized}\n</conversation>\n\n` + optional `<previous-summary>` block + prompt + optional `\n\nAdditional focus: {customInstructions}` (`packages/coding-agent/src/core/compaction/compaction.ts:717-733`) → [[pi--transcript-serialization-for-summary]].
- First compaction uses `SUMMARIZATION_PROMPT`; later ones `UPDATE_SUMMARIZATION_PROMPT` → [[pi--iterative-summary-update]]. Branch summaries use `BRANCH_SUMMARY_PROMPT` (same skeleton minus Critical Context) → [[pi--branch-summary]].
- Output cap `min(floor(0.8·reserveTokens), model.maxTokens)` (`packages/coding-agent/src/core/compaction/compaction.ts:712-715`); file lists appended after the LLM text (`packages/coding-agent/src/core/compaction/compaction.ts:1056-1058`) → [[pi--file-op-tracking]].

### `SUMMARIZATION_SYSTEM_PROMPT` (`packages/coding-agent/src/core/compaction/utils.ts:161-163`, verbatim)
```
You are a context summarization assistant. Your task is to read a conversation between a user and an AI assistant, then produce a structured summary following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary.
```

### `SUMMARIZATION_PROMPT` (`packages/coding-agent/src/core/compaction/compaction.ts:507-538`, verbatim)
```
The messages above are a conversation to summarize. Create a structured context checkpoint summary that another LLM will use to continue the work.

Use this EXACT format:

## Goal
[What is the user trying to accomplish? Can be multiple items if the session covers different tasks.]

## Constraints & Preferences
- [Any constraints, preferences, or requirements mentioned by user]
- [Or "(none)" if none were mentioned]

## Progress
### Done
- [x] [Completed tasks/changes]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues preventing progress, if any]

## Key Decisions
- **[Decision]**: [Brief rationale]

## Next Steps
1. [Ordered list of what should happen next]

## Critical Context
- [Any data, examples, or references needed to continue]
- [Or "(none)" if not applicable]

Keep each section concise. Preserve exact file paths, function names, and error messages.
```
Note: "The messages above" although the conversation is *inside* the same user message in `<conversation>` tags preceding the prompt (consistent: tags come first).

### Prompt timeline
| date | hash | change |
|---|---|---|
| 2025-12-02 | `5daef11b4` | plan doc surveys Claude Code / Codex / OpenCode prompts |
| 2025-12-04 | `6c2360af2` (#92) | v1: "You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task. Include: - Current progress and key decisions made - Important context, constraints, or user preferences - Absolute file paths of any relevant files that were read or modified - What remains to be done (clear next steps) - Any critical data, examples, or references needed to continue. Be concise, structured, and focused on helping the next LLM seamlessly continue the work." (sent as user message after real messages) |
| 2025-12-29 | `ac71aac09` | EXACT structured format (Goal / Constraints & Preferences / Progress Done-InProgress-Blocked / Key Decisions / Next Steps / Critical Context) + "Preserve exact file paths, function names, and error messages."; branch summary gets same format |
| 2025-12-29 | `a602e8aba` | Goal "[1-2 sentences: …]" → "[What is the user trying to accomplish? Can be multiple items if the session covers different tasks.]" |
| 2025-12-29 | `3c6c9e52c` | system prompt added ("…between a user and an AI **coding** assistant…"); prompts start "The messages above are…"; `reasoning:"high"` |
| 2026-06-07 | `72fd91135` (#5401) | "AI coding assistant" → "AI assistant" (CHANGELOG: "neutral AI assistant wording for non-coding agents") |
| 2026-09-30 | `ed0d6b91b` | durable copy adds "If the conversation starts with an earlier summary, preserve its information and fold the newer messages into it." |

## Constants
| name | value | path:line |
|---|---|---|
| summary `maxTokens` | 0.8 × 16384 = 13107 (clamped to model max) | `packages/coding-agent/src/core/compaction/compaction.ts:712-715` |

## Evidence commits
`5daef11b4` `6c2360af2` `ac71aac09` `a602e8aba` `3c6c9e52c` `72fd91135` `d35935200` `da6a81d39` `ed0d6b91b`

## Quirks
- Reasoning: originally forced `reasoning:"high"` (`3c6c9e52c`); now session thinking level only when `model.reasoning && level ≠ off` (`packages/coding-agent/src/core/compaction/compaction.ts:595-610`; `d35935200` #1793, `da6a81d39` #3438).
- Same anti-continuation system prompt reused for bug reports (`bug-report.ts:282-300`, `3c75b2747`).

## Durable variant (packages/durable)
- `packages/durable/src/harness/compaction.ts:59-98`: same system prompt; one `SUMMARIZATION_PROMPT` identical to coding-agent's plus the "fold an earlier summary" sentence; no update prompt, no branch prompt.

## Failures
[[summary-template-drops-goals]] · [[domain-biased-summarizer-prompt]] · [[summarizer-refusal]]
