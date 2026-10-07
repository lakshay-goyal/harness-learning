---
type: failure
concepts: [event-sourced-session-store, context-projection]
harnesses: [opencode]
---
**Symptom** History was ordered by wall-clock time or id, so messages could come back in the wrong order. A reorder in `filterCompacted` caused double auto-compaction (`94564f3588` 2026-05-14, #27545). The legacy loop then had to be explicitly ordered by time (`db581e47a3` 2026-08-07, #40990).

**Root cause** "Wall-clock timestamps may collide or move backwards, so they are not safe pagination boundaries" (`specs/v2/schema-changelog.md:378`).

**Fix · [[opencode]]** In the v2 runtime, projected messages carry their source aggregate `seq`, and ordering and pagination use `seq`, not timestamps (`specs/v2/session.md:175`; `specs/v2/schema-changelog.md:362-385`). The legacy runtime still orders by `time.created`, then id.

**Lesson** Use one monotonic ordering authority, such as a durable sequence number, for replay, context projection and pagination.

Related: [[event-sourced-session-store]] · [[context-projection]] · [[opencode--event-sourced-session-store|opencode]] · [[session-store-format]]
