---
type: concept
stage: state
tier: candidate
aliases: [CURRENT_SESSION_VERSION, migrateV1ToV2, migrateV2ToV3, migrateToCurrentVersion, "migrate(value, fromVersion)", migration_failed, prepareArguments (legacy shapes)]
harnesses: [pi]
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

## Implementations
- [[pi--session-migration|pi]] — `CURRENT_SESSION_VERSION = 3`; v1→v2 linear ids + `firstKeptEntryIndex→Id`; v2→v3 `hookMessage→custom`; file rewritten; durable per-definition `migrate`.

## Failures
- none recorded

## Related
[[session-tree]] · [[tool-argument-repair]] · [[durable-execution]] · [[branch-scoped-extension-state]]
