---
type: implementation
harness: opencode
concept: summary-validation
commit: ecc4916b5a
files: [packages/core/src/session/compaction.ts:184-190, packages/core/src/session/compaction.ts:199-230, packages/opencode/src/session/processor.ts:316-334, packages/opencode/src/session/compaction.ts:425-459, packages/opencode/src/session/compaction.ts:552]
---
[[summary-validation]] in [[opencode]].

## Mechanism

### Legacy runtime — prevent at source, little post-hoc checking
- Compaction step runs with `tools: {}` (`packages/opencode/src/session/compaction.ts:429`); the processor throws "Tool call not allowed while generating summary: <name>" on `tool-input-start`/`tool-call` for summary messages (`packages/opencode/src/session/processor.ts:316-334`).
- Overflowing compaction → error + stop (`compaction.ts:450-459`); any processor error → `return "stop"` before `Compacted` is published (`:552`).
- No length-stop or empty-text check found: a finished `summary: true` message with empty text is still a "completed compaction" whose `summaryText` is `undefined` (`compaction.ts:87-95`, `108`) (inference).

### v2 runtime — validate before the durable cutover
- Pre-flight: nothing old enough to summarize → skip (`packages/core/src/session/compaction.ts:184`); summary prompt estimate > `context − summaryOutput` → skip (`:189-190`).
- Stream: provider error event or `LLM.Error` → fail; `!summary.trim()` → fail (`:211-221`). Only then `Compaction.Ended` is published, which is the only event that activates the cut (`:222-229`; `specs/v2/session.md:117`). Failure leaves the previous boundary in place.
- `tools: []` on the request (`:207`).

## Constants
| name | value | path:line |
|---|---|---|
| v2 summary output cap | `min(output || 4096, 4096)` | `packages/core/src/session/compaction.ts:189` |

## Evolution
- 2026-06-05 `beae7290f3` v2 compaction with empty/error guards and the fit pre-check.

## Quirks / drift
- Neither runtime rejects a `length`-stopped summary explicitly; v2's 4096-token cap makes truncation plausible for long sessions ([[truncated-summary-persisted]] exposure, unverified).

## Failures
[[empty-compaction-summary]] · [[summarization-request-overflows]] · [[summarizer-emits-tool-calls]]

Contrast: [[pi--summary-validation|pi]] rejects error/length/tool-call summaries post hoc; opencode legacy prevents tool calls by throwing mid-stream, v2 adds empty-text and fit checks but no length check.
