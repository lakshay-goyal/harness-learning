---
type: failure
concepts: [summary-validation, auto-compaction]
harnesses: [pi]
---
**Symptom** — Summaries cut off at the output-token cap were saved as the session's compaction checkpoint; the missing half of the summary was permanently gone from model context.

**Root cause** — Only `stopReason === "error"` was rejected; a `length` stop returns partial text that looks valid.

**Fix · [[pi]]** — `97fa14e39` 2026-08-24 (#7048) `getSummarizationFailure` rejects `length`: "generation hit the token cap and the summary is incomplete"; doc: "A length stop contains partial text and must not become a session checkpoint." (`packages/coding-agent/src/core/compaction/compaction.ts:580-593`). Applied to history, turn-prefix and branch summaries. Durable requires a clean `stop` (`packages/durable/src/harness/compaction.ts:325-342`).

**Lesson** — A checkpoint replaces history: validate it like a disk write — clean stop, non-empty, no tool calls — before persisting.

Related: [[summary-validation]] · [[summary-output-budget-misfit]] · [[empty-compaction-summary]] · [[summarizer-emits-tool-calls]]
