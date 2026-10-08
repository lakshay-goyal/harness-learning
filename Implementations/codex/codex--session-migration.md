---
type: implementation
harness: codex
concept: session-migration
commit: 622e9e3696
files: [codex-rs/thread-store/src/local/rollout_migration.rs:1, codex-rs/cli/src/main.rs:219, codex-rs/state/src/sqlite.rs:34, codex-rs/state/migrations/0001_threads.sql:1, codex-rs/core/src/session/rollout_reconstruction.rs:522, codex-rs/core/src/context/world_state/mod.rs:301, codex-rs/features/src/lib.rs:425, codex-rs/config/src/config_toml.rs:804]
---
[[session-migration]] in [[codex]].

## Mechanism
- **Rollout format migration (Legacy → Paginated history)**: a crash-safe state machine — find rollouts, check eligibility, take maintenance + writer locks, canonicalize into a staged JSONL, project into SQLite, verify, atomically publish; "we always leave behind either the original legacy rollout or a recoverable paginated rollout. Once the rollout path is replaced, the durable `.pending` journal must be enough for a later migration run to finish SQLite recovery safely." (`codex-rs/thread-store/src/local/rollout_migration.rs:1-9`). Runs in background and via CLI `codex migrate-rollouts` (`codex-rs/cli/src/main.rs:219`, `:1524-1528`).
- **SQLite index migrations**: numbered SQL files per database — `codex-rs/state/migrations/` (60 files, `0001_threads.sql` → `0060_guardian_review_feedback.sql`), plus `thread_history_migrations/`, `memory_migrations/`, `logs_migrations/`, `goals_migrations/`, `queue_migrations/`. Breaking resets bump the DB file name instead (`state_5.sqlite`, `logs_2.sqlite`, `thread_history_1.sqlite` …, `codex-rs/state/src/sqlite.rs:34-39`); the index is rebuildable from rollouts → [[codex--sqlite-session-index|sqlite-session-index]].
- **Read-time shims (no rewrite)**:
  - legacy compactions without `replacement_history` rebuilt from user messages + summary, re-injecting canonical context (`codex-rs/core/src/session/rollout_reconstruction.rs:522-545`);
  - `ThreadRolledBack` markers still replayed after the API was removed (`codex-rs/core/src/context_manager/history.rs:755-880`);
  - world-state sections recognize legacy context fragments in migrated rollouts (`codex-rs/core/src/context/world_state/mod.rs:301-354`);
  - fork sanitization strips persisted hints "by kind, including persisted hints whose wording predates the current bundled instructions" (`6221a217e2`).
- **Config compatibility**: removed features stay parseable as no-ops ("Removed compatibility flag retained as a no-op so old configs can still parse `undo`", `codex-rs/features/src/lib.rs:425-428`; `Stage::Removed` "useless but kept for backward compatibility", `codex-rs/features/src/lib.rs:47-62`) → [[feature-flag-stages]]; `GhostSnapshotToml` "Legacy no-op setting retained for compatibility" (`codex-rs/config/src/config_toml.rs:804-814`); `OnFailure` approval mode a serde alias of on-request (`codex-rs/protocol/src/protocol.rs:1036`); legacy integer `fork_turns` accepted as `"all"` (`codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279-302`).
- **Hard breaks with migration message**: `wire_api = "chat"` / provider `ollama-chat` fail deserialization with `CHAT_WIRE_API_REMOVED_ERROR` (`codex-rs/model-provider-info/src/lib.rs:100-133`) → [[no-chat-completions-wire]].

## Evolution
- 2026-02-23 `eace7c6610` sqlite state lands (started `3878c3dc7c` 2026-01-28).
- 2026-05-26 `aad59a0916` (#24591) memory tables moved to a dedicated DB (`codex-rs/state/migrations/0035_drop_memory_tables.sql`); logs dropped from state DB (`codex-rs/state/migrations/0023_drop_logs.sql`).
- 2026-08-05 `6bb6e9045f` (#37175) legacy rollout migration to paginated history; 2026-08-06 `aac9f84247` (#37191) preserve legacy semantics; 2026-08-07 `4bb7ee3472` (#37348) migration tooling + background migration.

## Versus pi
- [[pi--session-migration]]: pi migrates eagerly on load (header version v1→v2→v3) and rewrites the file in place (crash window); codex stages + verifies + atomically publishes with a `.pending` journal, and keeps many read-time shims instead of rewriting.
