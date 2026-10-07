---
type: failure
concepts: [context-projection, auto-compaction, event-sourced-session-store]
harnesses: [opencode]
---
**Symptom** — Right after a compaction with a retained tail, the loop compacted a second time: it took the pre-compaction overflowing assistant as "the last finished message" and re-triggered. Later, imported/migrated sessions picked the wrong latest user/assistant message.

**Root cause** — `filterCompacted` reorders history to `[compaction user, summary, …retained tail…, …post-summary…]`, so array position is no longer chronological; the loop still took "last" by position. The first fix (max id) assumed ids are time-ordered, which is false for imported data.

**Fix · [[opencode]]**
- `811954880e` 2026-05-05 introduced the reorder (summary before tail).
- `94564f3588` 2026-05-14 (#27545) "prevent double auto-compaction from filterCompacted reorder": `MessageV2.latest()` by id.
- `db581e47a3` 2026-08-07 (#40990) "order legacy message loop by time": `isAfter` = `time.created`, id only as tie-breaker; comment "IDs are only a deterministic tie-breaker because imported messages do not necessarily have monotonic IDs" (`packages/opencode/src/session/message-v2.ts:591-617`).
- v2 avoids both: projected messages carry their aggregate `seq` and are ordered and paginated by it, because "Wall-clock timestamps may collide or move backwards, so they are not safe pagination boundaries" (`specs/v2/schema-changelog.md:378`; `packages/core/src/session/history.ts:49`).

**Lesson** — Once a projection reorders history, derive "latest" from one explicit ordering authority (a durable sequence), never from array position, id shape or wall-clock alone.

Related: [[context-projection]] · [[auto-compaction]] · [[event-sourced-session-store]] · [[time-ordered-id-prefix-collision]] · [[opencode--context-projection|opencode impl]]
