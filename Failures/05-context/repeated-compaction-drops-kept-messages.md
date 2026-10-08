---
type: failure
concepts: [iterative-summary-update, auto-compaction, compaction-cut-point, session-tree]
harnesses: [pi, opencode]
---
**Symptom** — After the second compaction in a session, messages that the first compaction had kept verbatim disappeared from both the model context and the new summary.

**Root cause** — Append-only log: the compaction entry is appended *after* the entries it keeps (they precede it in file order). The second compaction started its summarize range at "the entry after the previous compaction entry" in raw path order, so the previously-kept region was skipped entirely.

**Fix · [[pi]]** — `eeace7971` 2026-03-27 (#2608): range starts at the previous compaction's `firstKeptEntryId`, `tokensBefore` recomputed from the rebuilt context, compaction entries excluded from the summarized messages (`packages/coding-agent/src/core/compaction/compaction.ts:95`, `626-653` at that commit). HEAD expresses it over the canonical projection: `boundaryStart = prevCompactionIndex + 1` where the projection already places kept entries after the summary (`compaction.ts:883-895`, `466db0fec`).

**Fix · [[opencode]]** Variant on the first compaction: the "kept" tail was serialized and folded into the summary prompt ("Fold this serialized recent conversation tail into the summary") instead of staying verbatim, so the recent turns existed only as summary text. `811954880e` 2026-05-05 ordered the summary before the retained tail; `ca28dd02ec` 2026-05-12 (#27145) "restore tail turns after summarization": the tail start is persisted as `tail_start_id` on the compaction part and `filterCompacted` restores those messages after the summary (`packages/opencode/src/session/compaction.ts:461-466`; `packages/opencode/src/session/message-v2.ts:576-583`). Re-compaction hides earlier compaction pairs and passes the newest summary as `<prior-summary>` (`compaction.ts:363-366`).

**Lesson** — A compaction boundary is a (summary, firstKept) pair; the next summary must chain from `firstKept`, not from where the marker sits in the log.

Related: [[iterative-summary-update]] · [[auto-compaction]] · [[context-projection]] · [[pi--iterative-summary-update|pi impl]]
