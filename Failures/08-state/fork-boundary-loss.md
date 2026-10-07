---
type: failure
concepts: [session-fork, session-tree, auto-compaction]
harnesses: [pi, opencode]
---
**Symptom** — Forking a path containing labels orphaned subtrees (entries parented on dropped label nodes) (#5669); forking a compacted session lost the compaction boundary when `firstKeptEntryId` pointed at a label (#8989/#8990), so the fork's context rebuilt wrongly.

**Root cause** — Labels are real tree nodes (appended under the leaf, moving it) but `createBranchedSession` filtered them out while copying; references into the copied path (`parentId`, `firstKeptEntryId`) weren't remapped.

**Fix · [[pi]]**
- `adf567c1c` 2026-06-12 "rechain fork paths without labels": retained entries re-parented to the previous retained entry; labels recreated after the path (`packages/coding-agent/src/core/session-manager.ts:1639-1711`).
- `2631b25c3` 2026-09-03 (#8990) "preserve compaction boundary when forking": `firstKeptEntryId` mapped through `replacementByLabelId` to the next real entry (`:1650-1660`).

**Fix · [[opencode]]** Fork copies SQLite rows with fresh ids, so references inside the copy pointed at the source session. `a5edf3a311` 2026-01-09 (#6445) "resolve broken forked sessions with compactions due to missing parent-child message references": assistant `parentID` remapped through an old→new id map; `d71b827d8c` 2026-04-29 (#24898) "remap compaction tail_start_id when forking" (`packages/opencode/src/session/session.ts:703-727`).

**Lesson** — Forking must copy derived state (parent chains, boundary references, labels), not just entries; metadata stored as tree nodes forces reference remapping whenever a path is filtered.

Related: [[session-fork]] · [[session-tree]] · [[auto-compaction]] · [[fork-writes-wrong-file]] · [[branch-summary-records-wrong-source-leaf]] · [[pi--session-fork|pi]]
