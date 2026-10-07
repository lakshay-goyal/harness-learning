---
type: implementation
harness: codex
concept: sqlite-session-index
commit: 622e9e3696
files: [codex-rs/state/src/lib.rs:1, codex-rs/state/src/lib.rs:7, codex-rs/state/src/sqlite.rs:32, codex-rs/state/src/sqlite.rs:34, codex-rs/state/src/sqlite.rs:418, codex-rs/state/migrations/0001_threads.sql:1, codex-rs/state/thread_history_migrations/0001_thread_history.sql:1, codex-rs/thread-store/src/lib.rs:1, codex-rs/thread-store/src/local/rollout_migration.rs:1]
---
[[sqlite-session-index]] in [[codex]].

## Mechanism
- Crate doc: "SQLite-backed state for rollout metadata … extracts rollout metadata from JSONL rollouts and mirrors it into a local SQLite database. Backfill orchestration and rollout scanning live in `codex-core`." (`codex-rs/state/src/lib.rs:1-5`).
- **Engine guard**: compile-time assert bundled SQLite ≥ 3.51.3, "bundled SQLite must include the WAL-reset corruption fix" (`codex-rs/state/src/lib.rs:7-10`).
- **Files by concern** (`codex-rs/state/src/sqlite.rs:34-39`): `state_5.sqlite`, `logs_2.sqlite`, `goals_1.sqlite`, `memories_1.sqlite`, `queue_1.sqlite`, `thread_history_1.sqlite`; version suffix = breaking schema reset. Location override env `CODEX_SQLITE_HOME` (`codex-rs/state/src/lib.rs:137`); managed requirements can pin `sqlite_home` ([[codex--layered-settings|layered-settings]]). Busy timeout 5 s (`codex-rs/state/src/sqlite.rs:32`).
- **`threads` table** (`codex-rs/state/migrations/0001_threads.sql:1-25`): id, rollout_path, created/updated (ms), source, model_provider, cwd, title, sandbox/approval mode, tokens_used, has_user_event, archived, git sha/branch/origin; later migrations (0007–0056) add first_user_message, preview, name, model, reasoning_effort, agent nickname/role/path, memory_mode, history_mode, recency_at, is_pinned, section, project_id, originator, creator ids.
- **Other tables**: thread_dynamic_tools, thread_spawn_edges (sub-agent graph), thread_sections, projects, thread_attachments, guardian_review_feedback. No FTS index observed.
- **Paginated history projection** (`codex-rs/state/thread_history_migrations/0001_thread_history.sql:1-37`): `thread_turns(thread_id, turn_id, rollout_ordinal, status, error_json, timings, first_user_item_id, final_agent_item_id)`, `thread_items(thread_id, turn_id, item_id, rollout_ordinal, created_at_ms, item_json)`, `thread_history_projection_state(next_rollout_byte_offset, next_rollout_ordinal)` for incremental tailing of the JSONL. Backs app-server `thread/turns/list`, `thread/items/list`.
- **Legacy → paginated migration**: staged, verified, atomically published with `.pending` journal (`codex-rs/thread-store/src/local/rollout_migration.rs:1-9`) → [[codex--session-migration|session-migration]].
- **Storage boundary**: `ThreadStore` = storage-neutral persistence ("Implementations are responsible for resolving that id to local rollout files, RPC requests, or any other backing store", `codex-rs/thread-store/src/lib.rs:1-5`); durability fences via `PersistContext` (`codex-rs/thread-store/src/store.rs:68-94`).
- **Attachments**: `codex-rs/attachment-store` = "Storage-neutral attachment persistence interfaces" (`codex-rs/attachment-store/src/lib.rs:1`); thread attachment limits live in state (constants below); app-server `thread/attachment/{add,list,remove}` + `thread/attachmentOwner/list`.
- **Corruption handling**: auto-backup and rebuild from rollouts (`3691fe5b76`); `PRAGMA quick_check` once at open limited to 100 ms ("Limit startup validation to 100 ms.", `codex-rs/state/src/sqlite.rs:414-419`) → [[index-db-corruption]].
- **Job state**: memory stage-1 leases (`codex-rs/state/memory_migrations/0001_memories.sql:1-30`) → [[cross-session-memory]]; queued submissions ≤ 100 per thread (`codex-rs/state/src/lib.rs:113`) → [[follow-up-queue]]; goals in `goals_1.sqlite` → [[persistent-goal-continuation]].

## Constants
| name | value | path:line |
|---|---|---|
| `STATE_DB_FILENAME` | `state_5.sqlite` | codex-rs/state/src/sqlite.rs:38 |
| `THREAD_HISTORY_DB_FILENAME` | `thread_history_1.sqlite` | codex-rs/state/src/sqlite.rs:39 |
| `DEFAULT_BUSY_TIMEOUT` | 5 s | codex-rs/state/src/sqlite.rs:32 |
| startup `quick_check` budget | 100 ms | codex-rs/state/src/sqlite.rs:418-419 |
| min bundled SQLite | 3.51.3 | codex-rs/state/src/lib.rs:7-10 |
| `MAX_QUEUE_ITEMS` | 100 per thread | codex-rs/state/src/lib.rs:113 |
| `MAX_THREAD_ATTACHMENT_PAYLOAD_BYTES` | 64 KiB | codex-rs/state/src/lib.rs:116 |
| attachments per thread / list page | 100 / 100 | codex-rs/state/src/lib.rs:125-128 |
| `SQLITE_HOME_ENV` | `CODEX_SQLITE_HOME` | codex-rs/state/src/lib.rs:137 |

## Evolution
- 2026-01-28 `3878c3dc7c` sqlite state started; landed 2026-02-23 `eace7c6610`.
- 2026-04-14 `dae56994da` `ThreadStore` interface.
- 2026-05-26 `aad59a0916` (#24591) memory state to dedicated DB (`0035_drop_memory_tables.sql`); logs dropped into logs DB (`0023_drop_logs.sql`).
- 2026-06-10 `3691fe5b76` (#26859) auto-backup + rebuild from rollouts.
- 2026-08-05 `6bb6e9045f` paginated history projection / migration.
- 2026-09-30 `3620b2caf8` (#49701) bounded `quick_check` at open.

## Quirks
- Index is explicitly second-class: "the data is reconstructable from the rollouts on disk" (`3691fe5b76` body).

## Versus pi
- pi has no index; sessions are discovered by scanning JSONL headers with bounded reads ([[pi--session-tree]]). pi-durable's SQLite is a primary store, not a mirror ([[pi--durable-execution]]). See [[session-store-format]].
