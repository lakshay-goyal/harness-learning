---
type: failure
concepts: [iterative-summary-update, auto-compaction, compaction-cut-point, session-tree]
harnesses: [pi]
---
**Symptom** — After the second compaction in a session, messages that the first compaction had kept verbatim disappeared from both the model context and the new summary.

**Root cause** — Append-only log: the compaction entry is appended *after* the entries it keeps (they precede it in file order). The second compaction started its summarize range at "the entry after the previous compaction entry" in raw path order, so the previously-kept region was skipped entirely.

**Fix · [[pi]]** — `eeace7971` 2026-03-27 (#2608): range starts at the previous compaction's `firstKeptEntryId`, `tokensBefore` recomputed from the rebuilt context, compaction entries excluded from the summarized messages (`packages/coding-agent/src/core/compaction/compaction.ts:95`, `626-653` at that commit). HEAD expresses it over the canonical projection: `boundaryStart = prevCompactionIndex + 1` where the projection already places kept entries after the summary (`compaction.ts:883-895`, `466db0fec`).

**Lesson** — A compaction boundary is a (summary, firstKept) pair; the next summary must chain from `firstKept`, not from where the marker sits in the log.

Related: [[iterative-summary-update]] · [[auto-compaction]] · [[context-projection]] · [[pi--iterative-summary-update|pi impl]]
