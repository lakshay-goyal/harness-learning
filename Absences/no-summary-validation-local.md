---
type: absence
harnesses: [codex]
---
# no-summary-validation-local

Pre/mid-turn local compaction accepts an empty summary and does not check length-truncation or tool calls in the summary output.

**What's missing**
- Empty summary replaced by "(no summary available)" (`codex-rs/core/src/compact.rs:752-756`); only PostTurn requires a non-empty summary (`codex-rs/core/src/compact.rs:369-377`); remote compaction validates only "exactly one compaction item" (`codex-rs/core/src/compact_remote_v2.rs:475-480`).

**Evidence of decision**
- Observed design (M6); post-turn requirement added with `49e248d4c3` 2026-09-18.

**Implication**
- A failed summarization silently shrinks context to user messages only; contrast pi's [[summary-validation]].

Related: [[summary-validation]] · [[auto-compaction]] · [[no-structured-compaction-template]] · [[Absences]]
