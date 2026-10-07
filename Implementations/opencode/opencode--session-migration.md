---
type: implementation
harness: opencode
concept: session-migration
commit: ecc4916b5a
files: [packages/core/src/database/database.ts:22-37, packages/opencode/src/storage/storage.ts:81-240, packages/opencode/src/session/message-v2.ts:591-617, packages/core/src/session/context-epoch.ts:56-58]
---
[[session-migration]] in [[opencode]].

## Mechanism
- **SQL migrations** (drizzle): `DatabaseMigration.apply(db)` runs on every open after the PRAGMAs (`packages/core/src/database/database.ts:22-37`); 38 timestamped files in `packages/core/src/database/migration/` at HEAD.
- **Legacy JSON storage** (`<data>/storage/**.json`): numbered `MIGRATIONS` with a `migration` marker file; each step run once, stop on first failure (`packages/opencode/src/storage/storage.ts:81-240`). Still used for `session_diff` blobs.
- **JSON → SQLite** move: `6d95f0d14c` 2026-02-13 "sqlite again" added `json-migration.ts` (437 lines); removed 2026-06-02 `ca2acc4f8d` "remove JSON storage migration" — pre-February sessions not migrated by then are no longer imported (inference).
- **Lenient decode of stored history**: after the Effect Schema migration, stored parts with negative token counts, non-finite timestamps, missing patch fields or legacy summary diffs were rejected; schemas loosened (`c6e6bdf59f`, `29250a0efb`, `16ddf5f559`, `d62442bb5d`, `d373c562f2`, all 2026-05) → [[strict-schema-rejects-legacy-records]].
- **Ordering of migrated data**: imported messages do not have monotonic ids, so "latest" is chosen by `time.created` then id (`packages/opencode/src/session/message-v2.ts:591-617`, `db581e47a3`).

### v2 runtime
- Pre-release v2 tables are disposable ("experimental V2 event history containing the retired event is disposable", `specs/v2/schema-changelog.md:26`): incompatible iterations reset experimental events/projections/inputs/epochs while preserving canonical V1 `session`/`message`/`part` rows (migrations `20260622170816_reset_v2_session_state`, `20260622202450_simplify_session_input`; `specs/v2/schema-changelog.md:10-26`).
- Event payloads are encoded before storage and decoded on replay "so schema transforms remain explicit at the durable boundary" (`specs/v2/schema-changelog.md:81`); stored Context Snapshot decode failure → typed `ContextSnapshotDecodeError` instead of a defect (`packages/core/src/session/context-epoch.ts:56-58`, `a261b55e43`).
- Known gap: migration claiming across processes is "protected only by an in-process semaphore, so two processes starting against one SQLite database can still race" (`specs/v2/todo.md:123-125`).

## Constants
| name | value | path:line |
|---|---|---|
| SQL migration files | 38 | `packages/core/src/database/migration/` |

## Evolution
- 2026-02-13 `6d95f0d14c` (#10597) SQLite storage + JSON migration.
- 2026-05-09 `c6e6bdf59f` / `29250a0efb` tolerate legacy numeric data; `d62442bb5d` optional patch for migrated sessions.
- 2026-06-02 `ca2acc4f8d` JSON migration removed.
- 2026-06-05 `a261b55e43` recoverable snapshot decode; 2026-06-22 v2 state reset migrations.
- 2026-08-07 `db581e47a3` order legacy loop by time; 2026-08-13 `d8bf79225f` preserve v1 database compatibility → [[projector-depends-on-transitional-table]].

## Quirks / drift
- Two runtimes share one database: v1 code must tolerate v2 tables being absent and v2 projectors must not assume them.

Contrast: [[pi--session-migration|pi]] versions the JSONL header and rewrites the file on load (v1→v3); opencode relies on SQL migrations plus lenient decoders, and treats v2 experimental state as resettable.
