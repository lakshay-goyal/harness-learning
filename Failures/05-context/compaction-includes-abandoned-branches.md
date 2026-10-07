---
type: failure
concepts: [auto-compaction, session-tree]
harnesses: [pi]
---
**Symptom** — In branched sessions compaction summarized entries from abandoned branches, producing token-overflow errors and summaries of work not on the current path.

**Root cause** — Compaction read all session entries instead of the active root→leaf path.

**Fix · [[pi]]** — `ddda8b124` 2025-12-31 "fix compaction for branched sessions": use `sessionManager.getPath()` / `getBranch()` (current branch only) (`packages/coding-agent/src/core/agent-session.ts` `_runAutoCompaction` → `prepareCompaction(this.sessionManager.getBranch(), …)` at HEAD); CHANGELOG 0.31.0.

**Lesson** — In a tree-shaped session every context operation (estimate, cut, summarize) must walk only the active path.

Related: [[auto-compaction]] · [[session-tree]] · [[branch-summary]] · [[context-projection]]
