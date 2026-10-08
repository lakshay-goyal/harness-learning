---
type: implementation
harness: pi
concept: summary-validation
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/compaction.ts:584, packages/coding-agent/src/core/compaction/compaction.ts:759, packages/coding-agent/src/core/compaction/compaction.ts:876, packages/coding-agent/src/core/compaction/branch-summarization.ts:356, packages/durable/src/harness/compaction.ts:325]
---
[[summary-validation]] in [[pi]].

## Mechanism
- **Pre-flight** `prepareCompaction` returns `undefined` when the last path entry is already a compaction (`packages/coding-agent/src/core/compaction/compaction.ts:876-878`) or both ranges are empty (`packages/coding-agent/src/core/compaction/compaction.ts:914`); auto-compaction returns before emitting `compaction_start` (`7d08c81a0`); manual `/compact` throws "Already compacted" / "Nothing to compact (session too small)".
- **`getSummarizationFailure(response, label)`** (`packages/coding-agent/src/core/compaction/compaction.ts:584-593`): `error` → "`<label>` failed: <errorMessage>"; `length` → "`<label>` failed: generation hit the token cap and the summary is incomplete". Doc comment: "A length stop contains partial text and must not become a session checkpoint."
- **Tool-call rejection**: "Summarization attempted to call a tool" (`packages/coding-agent/src/core/compaction/compaction.ts:759-761`), "Turn prefix summarization attempted to call a tool" (`packages/coding-agent/src/core/compaction/compaction.ts:1111-1113`), "Branch summarization attempted to call a tool" (`packages/coding-agent/src/core/compaction/branch-summarization.ts:363-365`).
- Compaction throws (caught by `_runAutoCompaction` → `compaction_end{errorMessage}`; history untouched); branch summary returns `{aborted:true}` on abort, `{error}` on failure (`packages/coding-agent/src/core/compaction/branch-summarization.ts:356-365`), `"No content to summarize"` when nothing fits, `summary || "No summary generated"`.
- Transient errors are retried *before* validation by `retryAssistantCall` (`packages/coding-agent/src/core/compaction/compaction.ts:619-639`); deterministic errors and aborts return immediately.

## Evolution
- 2025-12-31 (0.31.0) throw on LLM errors instead of returning "" (CHANGELOG, hash unverified).
- 2026-06-18 `7d08c81a0` (#4811) avoid empty compaction summaries.
- 2026-08-17 `90305d90a` disable tools during summarization (`toolChoice:"none"`) + reject tool calls.
- 2026-08-24 `97fa14e39` (#7048) reject `stopReason=length` summaries.
- 2026-08-26 `6b36eb592` (#8649, #8638) remove explicit tool choice; keep post-hoc rejection.

## Evidence commits
`7d08c81a0` `90305d90a` `97fa14e39` `6b36eb592`

## Quirks
- Coding-agent does **not** reject an empty-text `stop` response (no check after `contentText`, `packages/coding-agent/src/core/compaction/compaction.ts:763-765`) — an empty summary would persist (inferred); durable does reject it.
- No content-level checks (e.g. required sections present).

## Durable variant (packages/durable)
- `summaryText` (`packages/durable/src/harness/compaction.ts:325-333`): only clean `stop`, no toolCall, non-empty trimmed text; `summaryFailure` (`:335-342`) messages: "Summarization failed: …", "Summarization hit the token limit; the summary is incomplete", "Summarization attempted to call a tool", "Summarization produced no text". Retryable errors → `retry` phase; others terminal `failed`.

## Failures
[[truncated-summary-persisted]] · [[summarizer-emits-tool-calls]] · [[empty-compaction-summary]]
