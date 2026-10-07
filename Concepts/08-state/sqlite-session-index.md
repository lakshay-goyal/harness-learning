---
type: concept
stage: state
tier: variant
aliases: [codex-state, state_5.sqlite, thread_history_1.sqlite, CODEX_SQLITE_HOME, ThreadStore, LocalThreadStore, backfill_state, thread_history_projection_state, session-index-db]
harnesses: [codex]
---
Append-only transcript files stay the source of truth while a local SQLite database mirrors per-thread metadata (and optionally a paginated item projection) for listing, search, pagination and background jobs; the database is disposable — it can be backfilled or rebuilt from the files.

## Why
- Listing/searching thousands of sessions by scanning JSONL headers is slow; UIs (resume picker, desktop sidebar, app-server `thread/list`) need indexed queries, pagination and per-thread flags (pinned, archived, project).
- Background jobs (memory extraction leases, queued submissions, goals) need transactional state that a log file cannot give.
- A second store is a second failure surface: corruption must not lose conversations ([[index-db-corruption]]); keeping the index derivable makes it safe to delete.

## Design space
- **No index, scan files** (pi: header discovery with bounded scans, [[pi--session-tree]]) vs **SQLite mirror of the log** (codex) vs **DB as primary store** (pi-durable SQLite storage, [[durable-execution]]).
- **Scope of mirror**: metadata only (`threads` table) vs metadata + item projection for paginated history (codex `thread_history_1.sqlite`, incremental tailing by byte offset/ordinal).
- **One DB vs split by concern**: codex splits state / logs / goals / memories / queue / thread_history into separate files so schema resets and contention are isolated.
- **Schema evolution**: numbered SQL migrations + file-name version bump for breaking resets (codex) vs rebuild-on-mismatch.
- **Corruption policy**: fail vs auto-backup + rebuild from logs (codex) + bounded `quick_check` at open + minimum engine version assert.
- **Full-text search**: none observed in codex (no FTS) vs FTS index.
- **Storage boundary**: concrete files vs storage-neutral trait so ids can resolve to local files, RPC, or other backing stores (codex `ThreadStore`).

## Implementations
- [[codex--sqlite-session-index|codex]] — `codex-rs/state`: `state_5.sqlite` `threads` table mirrored from rollouts, `thread_history_1.sqlite` paginated projection, separate goals/memories/queue/logs DBs; backup+rebuild on corruption; `ThreadStore` trait.

## Failures
- [[index-db-corruption]]

## Related
[[session-tree]] · [[session-migration]] · [[cross-session-memory]] · [[cross-session-prompt-history]] · [[follow-up-queue]] · [[persistent-goal-continuation]] · [[client-server-session-split]] · [[session-store-format]]
