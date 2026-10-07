---
type: failure
concepts: [summary-validation, auto-compaction]
harnesses: [pi]
---
**Symptom** — The UI showed compaction start/end with nothing compacted, and empty summaries were produced for sessions with no eligible messages; earlier, LLM errors returned "" as the summary.

**Root cause** — Preparation returned a work item with empty ranges, and start events were emitted before checking; error responses weren't thrown.

**Fix · [[pi]]** — `7d08c81a0` 2026-06-18 (#4811): `prepareCompaction` returns `undefined` when there is nothing to summarize (`packages/coding-agent/src/core/compaction/compaction.ts:914`), auto-compaction returns before `compaction_start` (`packages/coding-agent/src/core/agent-session.ts:3110-3118`), manual `/compact` reports "Nothing to compact (session too small)". 0.31.0: throw on LLM errors instead of returning "" (hash unverified). Gap: coding-agent still doesn't reject an empty-text `stop`; durable does ("Summarization produced no text").

**Lesson** — Decide "is there work?" before announcing work, and treat empty output as failure.

Related: [[summary-validation]] · [[auto-compaction]] · [[truncated-summary-persisted]]
