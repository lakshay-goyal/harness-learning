---
type: implementation
harness: pi
concept: session-migration
commit: b30a6dd77
files: [packages/coding-agent/src/core/session-manager.ts:41, packages/coding-agent/src/core/session-manager.ts:286, packages/coding-agent/src/core/session-manager.ts:316, packages/coding-agent/src/core/session-manager.ts:337, packages/coding-agent/src/core/session-manager.ts:1092, packages/coding-agent/src/migrations.ts:1, packages/durable/docs/spec.md:990]
---
[[session-migration]] in [[pi]].

## Mechanism
- `CURRENT_SESSION_VERSION = 3` (`packages/coding-agent/src/core/session-manager.ts:41`); header `version?` absent in v1 (`:45`).
- **v1 → v2** `migrateV1ToV2` (`:286-313`): assign ids (`generateId`) and `parentId = previous entry` linearly; convert compaction `firstKeptEntryIndex` → `firstKeptEntryId` (`:301-310`). Introduced with the tree (`c58d5f20a`, 2025-12-25).
- **v2 → v3** `migrateV2ToV3` (`:315-335`): rename message role `hookMessage` → `custom` (hooks merged into extensions, `c6fc08453`, 2026-01-05, #454).
- `migrateToCurrentVersion` runs needed steps by header version (`:337-347`); on open, if any migration ran the **whole file is rewritten** (`:1092-1094`) via `_rewriteFile` = `openSync(path, "w")` + write each entry + close (`:1124-1134`) — in place, no temp+rename, no fsync.
- **Read-time normalization** (no version bump): null message content normalized at load (`:439-452`; `8c0ccd14b` #6259/#6276); legacy tool-call argument shapes repaired by per-tool `prepareArguments` before validation (`packages/agent/src/agent-loop.ts:694-706`; `b5f425ad1`, 2026-03-29 — resumed sessions broke after edit-schema change) → [[tool-argument-repair]].
- **Out-of-file migrations** `packages/coding-agent/src/migrations.ts` (`cb6310e15`, 2025-12-26, #320): sessions misplaced in `~/.pi/agent/*.jsonl` by a v0.30.0 bug auto-moved on startup; legacy auth migration moved here.
- Fork writes `version: CURRENT_SESSION_VERSION` on new files (`:1675`).

## Constants
| name | value | path:line |
|---|---|---|
| `CURRENT_SESSION_VERSION` | 3 | packages/coding-agent/src/core/session-manager.ts:41 |
| durable JSONL `FORMAT_VERSION` | 1 | packages/durable/src/storage/jsonl/storage.ts:28 |

## Evolution
- 2025-12-25 `c58d5f20a` v2 (tree).
- 2025-12-26 `cb6310e15` startup migrations module.
- 2026-01-05 `c6fc08453` v3.
- 2026-03-29 `b5f425ad1` `prepareArguments` legacy-shape shim.
- 2026-07-06 `8c0ccd14b` null-content normalization at ingestion boundaries.
- 2026-09-24 `5d4de953c` durable checkpoints + document migrations.

## Evidence commits
`c58d5f20a`, `cb6310e15`, `c6fc08453`, `b5f425ad1`, `8c0ccd14b`, `5d4de953c`

## Quirks
- Non-atomic in-place rewrite during migration: crash mid-rewrite can truncate the session (inferred from `_rewriteFile`).
- No version bump for semantic additions (`context_edit`, `usage`, mid-conversation `SystemMessage`): older pi versions silently skip/mishandle unknown entry types (unverified).

## Durable variant (packages/durable)
- Definitions (documents, tasks) carry `version` and optional `migrate(value, fromVersion)` (`packages/durable/docs/spec.md:990,1100-1108`); migration is **lazy on typed access** — "Open does not scan or migrate other documents" (`spec.md:627-628`).
- A task whose definition is missing/unmigratable is never terminalized: stays pending/waiting, `blocked: missing_task | task_too_old | migration_failed` (`spec.md:523,624-627`; `src/harness/scheduler.ts:57`); `ready{migrates}` shown in inspection (`spec.md:921-923`).
- Document deltas cannot cross a version boundary (`spec.md:4513`). SQLite schema migrations in `src/storage/sqlite/migrations.ts:9-87`.

## Failures
- none recorded
