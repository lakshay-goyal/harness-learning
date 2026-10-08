---
type: implementation
harness: pi
concept: transcript-serialization-for-summary
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/utils.ts:94, packages/coding-agent/src/core/compaction/utils.ts:114, packages/coding-agent/src/core/compaction/utils.ts:161, packages/coding-agent/src/core/compaction/compaction.ts:619, packages/coding-agent/src/core/compaction/compaction.ts:681, packages/coding-agent/src/core/messages.ts:148, packages/durable/src/harness/compaction.ts:351]
---
[[transcript-serialization-for-summary]] in [[pi]].

## Mechanism
- Pipeline: `convertToLlm(messages)` first (bashExecution / custom / summaries → user text, `packages/coding-agent/src/core/messages.ts:148-196`) → `serializeConversation(llmMessages)` (`packages/coding-agent/src/core/compaction/utils.ts:114-155`) → wrapped in `<conversation>` tags (compaction, branch) or `# Conversation` heading (turn prefix) → single **user** message under `SUMMARIZATION_SYSTEM_PROMPT` via `buildSummarizationContext` + `normalizeContext` (`packages/coding-agent/src/core/compaction/compaction.ts:681-693`).
- Serialization format, blocks joined by blank lines:
  - `[User]: <text>` (empty skipped; images dropped — only text extracted, `packages/coding-agent/src/core/compaction/utils.ts:119`)
  - `[Assistant thinking]: <all thinking blocks joined \n>`
  - `[Assistant]: <text>` (only if a text block exists)
  - `[Assistant tool calls]: name(k=<JSON v>, …); name2(…)` (full args JSON)
  - `[Tool result]: <≤2000 chars>` + `\n\n[... N more characters truncated]` (`truncateForSummary`, head kept, `packages/coding-agent/src/core/compaction/utils.ts:96-104`)
  - system messages never serialized (excluded upstream: "System messages are prompt state, not conversation; the compaction entry carries their replay", `packages/coding-agent/src/core/compaction/compaction.ts:98-102`).
- Source comment: "This prevents the model from treating it as a conversation to continue." (`packages/coding-agent/src/core/compaction/utils.ts:107-113`); `generateSummaryWithUsage`: "Serialize conversation to text so model doesn't try to continue it" (`packages/coding-agent/src/core/compaction/compaction.ts:723-726`).
- **No tools** sent in the summarization context; any `toolCall` in the response is rejected (`packages/coding-agent/src/core/compaction/compaction.ts:759-761`) → [[pi--summary-validation]].
- `completeSummarization` (`packages/coding-agent/src/core/compaction/compaction.ts:619-639`): `cacheRetention:"none"`, `sessionId ?? uuidv7()`, `retryAssistantCall` with session retry policy; through session `streamFn` if present.
- `SUMMARIZATION_SYSTEM_PROMPT` verbatim in [[pi--structured-compaction-summary]]; reused for bug reports (`packages/coding-agent/src/core/bug-report.ts:282-300`, `3c75b2747`).

## Constants
| name | value | path:line |
|---|---|---|
| `TOOL_RESULT_MAX_CHARS` | 2000 | `packages/coding-agent/src/core/compaction/utils.ts:94` |
| durable `TOOL_RESULT_MAX_CHARS` | 2000 (mirror) | `packages/durable/src/harness/compaction.ts:55` |

## Evolution
- 2025-12-04 `6c2360af2` v1 summarized by sending real messages + a final user instruction.
- 2025-12-29 `3c6c9e52c` system prompt "Do NOT continue the conversation…"; `2add465fb` "Instead of passing conversation as LLM messages (which makes the model try to continue it), serialize to text wrapped in <conversation> tags." Tool args sliced to 100 chars, results to 1000.
- 2025-12-30 `17ce3814a` `convertToLlm` first, include `[Assistant thinking]`, **remove truncation** ("already token-budgeted").
- 2026-01-14 `72e29bce2` (#738) turn prefix serialized too.
- 2026-03-06 `c950c692a` (#1796) **re-add** 2000-char tool-result cap — summarization requests exceeded context limits (reversal of `17ce3814a`).
- 2026-06-07 `72fd91135` (#5401) "AI coding assistant" → "AI assistant".
- 2026-08-17 `90305d90a` `toolChoice:"none"` + reject; 2026-08-26 `6b36eb592` (#8649, #8638) forced toolChoice removed, rejection kept (gateways rejected `tool_choice` without tools, cf. `fe37e9f9b` #8607 adapter-side workaround reverted).
- 2026-09-22 `d192bd6dc` turn prefix moves to `# Conversation`/`# Instructions` headings.

## Evidence commits
`6c2360af2` `3c6c9e52c` `2add465fb` `17ce3814a` `72e29bce2` `c950c692a` `72fd91135` `90305d90a` `6b36eb592` `fe37e9f9b` `d192bd6dc` `3c75b2747`

## Quirks
- Tool-call *arguments* are never truncated (a huge `write` content arg is serialized in full) — only results are capped (inferred from `packages/coding-agent/src/core/compaction/utils.ts:131-137`); a session dominated by large writes could still overflow the summarizer (unverified).
- Truncation keeps the head of tool results, while bash results themselves were tail-truncated for the model — summarizer may see a different slice than the agent did.

## Durable variant (packages/durable)
- `serializeConversation` re-implemented (`packages/durable/src/harness/compaction.ts:351-380`): same tags; system messages omitted; previous summary (head marker) appears as `[User]: The conversation history before this point was compacted…`.

## Failures
[[summarizer-continues-conversation]] · [[summarization-request-overflows]] · [[summarizer-emits-tool-calls]] · [[domain-biased-summarizer-prompt]]
