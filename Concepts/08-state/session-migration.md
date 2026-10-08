---
type: concept
stage: state
tier: must-have
aliases: [CURRENT_SESSION_VERSION, migrateV1ToV2, migrateV2ToV3, migrateToCurrentVersion, "migrate(value, fromVersion)", migration_failed, prepareArguments (legacy shapes), DatabaseMigration, json-migration, reset_v2_session_state, ContextSnapshotDecodeError, rollout_migration, migrate-rollouts, thread_history_migrations, "Stage::Removed"]
harnesses: [pi, opencode, codex]
---
Versioned, on-load upgrade of the persisted session format (and of stored typed state) so old logs keep loading after format changes, plus shims for legacy shapes embedded in history (e.g. old tool-call argument schemas).

## Why
- Session logs outlive releases; resumes of months-old sessions must work after tree ids, role renames, or tool-schema changes (pi needed `prepareArguments` shims after edit-schema changes, `b5f425ad1`).
- Rewriting during migration is itself a crash window (non-atomic in pi coding-agent).

## Design space
- **Header version + eager full migration on load, then rewrite file** (pi coding-agent, v1→v2→v3).
- **Per-document/per-task definition versions with lazy migration on typed access** (pi-durable `migrate(value, fromVersion)`); unmigratable task stays pending, `blocked: migration_failed`, never terminalized.
- **Read-time compatibility shims** without rewriting (pi `prepareArguments` for legacy tool-call shapes; null-content normalization on load).
- **Rewrite strategy**: in-place truncate+write (pi) vs temp+rename.
- **Forward compat**: unknown future types dropped (pi-ai model catalog) vs rejected.
- **Staged + verified + atomic publish with recovery journal** ✔ codex (legacy → paginated rollouts; `codex migrate-rollouts` + background run).
- **Removed-but-parseable config keys** as no-ops (`Stage::Removed` features, legacy settings) ✔ codex; hard error with migration message for removed wire protocols ✔ codex.
- **Index schema**: numbered SQL migrations + file-name version bump for breaking resets ✔ codex ([[sqlite-session-index]]).
- **SQL schema migrations on open + lenient decoders for stored JSON** (opencode): strict schemas only on write, loosened for legacy numeric/timestamp/diff shapes.
- **Storage-format jump with a time-boxed importer**: JSON files → SQLite importer shipped 2026-02, deleted 2026-06 (opencode).
- **Reset, don't migrate, pre-release state**: experimental v2 events/projections/epochs reset while canonical v1 rows are preserved (opencode).
- **Two runtimes on one schema**: shared projectors must only touch tables both runtimes guarantee (opencode) → [[projector-depends-on-transitional-table]].

## Implementations
- [[pi--session-migration|pi]] — `CURRENT_SESSION_VERSION = 3`; v1→v2 linear ids + `firstKeptEntryIndex→Id`; v2→v3 `hookMessage→custom`; file rewritten; durable per-definition `migrate`.
- [[codex--session-migration|codex]] — crash-safe staged migration of legacy rollouts into paginated history (`.pending` journal, verify, atomic publish), numbered SQLite migrations + versioned DB file names, many read-time shims and no-op legacy config keys.
- [[opencode--session-migration|opencode]] — drizzle SQL migrations applied on every DB open (38 files), legacy JSON storage migrations with a marker file, lenient stored-history schemas, v2 experimental state reset migrations.

## Failures
- [[strict-schema-rejects-legacy-records]]
- [[projector-depends-on-transitional-table]]
- none recorded
- [[torn-log-tail-fuses-next-entry]] (08-state) — After a crash mid-write left a session JSONL without a trailing newline, the next appended entry was…

## Related
[[session-tree]] · [[tool-argument-repair]] · [[durable-execution]] · [[branch-scoped-extension-state]] · [[feature-flag-stages]] · [[session-store-format]]
