---
type: failure
concepts: [session-tree, partial-message-persistence]
harnesses: [pi]
---
**Symptom** — Exiting (Ctrl+C, crash) during the first turn of a new session lost the session entirely, including the user's prompt (#10000).

**Root cause** — To avoid empty session files from "open pi and quit", file creation was deferred until the first user+assistant exchange (`812f2f43c`, 2025-11-12), later "first assistant message" (`184c64833`, 2025-12-22); everything before the first response lived only in memory.

**Fix · [[pi]]** — `ff72faba2` 2026-09-25 "save new session file at the first user message": `_hasConversation()` true once a user **or** assistant message exists; first write `openSync(…,"wx")` flushes buffered setup entries (`packages/coding-agent/src/core/session-manager.ts:1160-1189`): "Starting at the user message (not the first assistant reply) keeps the prompt on disk if the first turn never completes (#10000)". Same rule reused by fork (`createBranchedSession`, `:1713-1724`).

**Lesson** — Defer creating empty logs, but persist user input before the first model call — it is the most expensive thing to lose.

Related: [[session-tree]] · [[partial-message-persistence]] · [[fork-writes-wrong-file]] · [[pi--session-tree|pi]]
