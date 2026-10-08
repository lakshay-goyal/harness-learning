---
type: failure
concepts: [branch-summary, session-tree]
harnesses: [pi]
---
**Symptom** — `branch_summary.fromId` pointed at the navigation destination instead of the abandoned branch's leaf, so tooling couldn't tell which branch a summary described.

**Root cause** — Wrong variable recorded after the leaf had already moved.

**Fix · [[pi]]** — `d711bd5f0` 2026-08-19 "Record the pre-navigation leaf in branch summary fromId instead of the destination node." — `branchWithSummary` captures `fromId = this.leafId ?? "root"` before moving the leaf (`packages/coding-agent/src/core/session-manager.ts:1600-1625`).

**Lesson** — Capture "from" state before mutating the pointer it's derived from.

Related: [[branch-summary]] · [[session-tree]] · [[branch-summary-wrong-common-ancestor]]
