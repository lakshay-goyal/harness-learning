---
type: failure
concepts: [branch-summary, session-tree]
harnesses: [pi]
---
**Symptom** — Branch summaries on `/tree` navigation covered far too much history: the common ancestor was always the root.

**Root cause** — `getBranch(targetId)` returns a root-first path; iterating it forward returned the first shared node — the root — instead of the deepest shared node.

**Fix · [[pi]]** — `92947a3dc` 2025-12-29 "Fix common ancestor finding: iterate backwards to find deepest ancestor" (`packages/coding-agent/src/core/compaction/branch-summarization.ts:118-127`).

**Lesson** — Deepest common ancestor = scan the target path from the leaf end; test with branches that share a long prefix.

Related: [[branch-summary]] · [[session-tree]] · [[branch-summary-records-wrong-source-leaf]]
