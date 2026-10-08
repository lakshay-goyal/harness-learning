---
type: failure
concepts: [session-migration, transcript-carried-system-prompt]
harnesses: [opencode]
---
**Symptom** — After a validator change, old sessions failed to load or crashed the session: stored parts with negative token counts, non-finite timestamps, missing patch fields or legacy summary diffs were rejected; in v2 a stored Context Snapshot that no longer matched the schema killed the session with a defect.

**Root cause** — Stored history was decoded with the same strict schema used for writes; data written by older versions (or by bugs) is valid history but invalid under today's schema, and `Effect.orDie` turned decode errors into unrecoverable defects.

**Fix · [[opencode]]**
- 2026-05 legacy loosening: `16ddf5f559` 2026-05-01 finite archived timestamp schema; `c6e6bdf59f` 2026-05-09 tolerate negative token counts; `29250a0efb` 2026-05-09 loosen remaining stored numeric schemas; `d62442bb5d` 2026-05-09 optional patch field for migrated sessions; `d373c562f2` 2026-05-09 accept legacy summary diffs.
- `a261b55e43` 2026-06-05 (#30905) `mapError` instead of `orDie` for context snapshot decoding → typed `ContextSnapshotDecodeError` (`packages/core/src/session/context-epoch.ts:56-58`); reconcile treats an undecodable stored source value as `Incompatible` → full baseline replacement (`packages/core/src/system-context/index.ts:154-157`, `packages/core/src/system-context/index.ts:238-239`).
- `1787fa4261` 2026-06-22 (#33404) `reset_v2_session_state` migration discards experimental v2 state wholesale — deletes context epochs, inputs, messages, events, event sequences and workspaces (`packages/core/src/database/migration/20260622170816_reset_v2_session_state.ts:8-14`).

**Lesson** — Decode stored history leniently and validate strictly only on write; a persisted prompt snapshot that no longer decodes should trigger a re-snapshot, not a crash.

Related: [[session-migration]] · [[transcript-carried-system-prompt]] · [[projector-depends-on-transitional-table]] · [[opencode--session-migration|opencode impl]]
