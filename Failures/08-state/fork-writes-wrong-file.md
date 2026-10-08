---
type: failure
concepts: [session-fork, session-tree]
harnesses: [pi]
---
**Symptom** — `/fork` appended the forked session's entries into the **parent** file (#1242); forking from a point before any assistant message wrote duplicate session headers/entries (#1672); in-memory forks taken before the active turn settled captured an inconsistent state (#8937).

**Root cause** — Fork changed identity (session id/path) and the "flushed" flag in separate steps from the writes; file-creating code paths had different rules for when to flush buffered entries; fork didn't wait for the turn to settle.

**Fix · [[pi]]**
- `13ac63c3c` 2026-02-04 "fork writes to new session file, not parent (fixes #1242)".
- `2f64df1e5` 2026-02-27 prevent duplicate headers when forking from a pre-assistant entry (#1672).
- `ff72faba2` 2026-09-25 — one `_hasConversation()` rule shared by `_persist` and `createBranchedSession` (`packages/coding-agent/src/core/session-manager.ts:1160-1170,1713-1724`).
- #8937 (`packages/coding-agent/CHANGELOG.md:540`, hash unverified) — in-memory forks wait for the turn to settle; runtime fork tears down (abort → persist → dispose) first (`agent-session-runtime.ts:167-178,262-350`).

**Lesson** — Fork = new identity + new file, set before any write; share one write-path rule across every file-creating path; settle the turn before snapshotting.

Related: [[session-fork]] · [[session-tree]] · [[session-lost-before-first-response]] · [[pi--session-fork|pi]]
