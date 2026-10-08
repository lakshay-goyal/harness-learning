---
type: implementation
harness: opencode
concept: event-sourced-session-store
commit: ecc4916b5a
files: [packages/core/src/database/database.ts:22-55, packages/opencode/src/session/session.ts:877-885, packages/core/src/session/projector.ts:242-260, packages/core/src/event.ts:118-124, packages/core/src/event.ts:205-360, packages/schema/src/session-event.ts:38-49, packages/schema/src/session-event.ts:209, packages/schema/src/session-event.ts:448-477, packages/opencode/src/event-v2-bridge.ts:51-55]
---
[[event-sourced-session-store]] in [[opencode]].

## Mechanism
- **Database**: one global SQLite file `opencode.db` (or `opencode-<channel>.db` for non-release channels, `OPENCODE_DB` override) in the data dir; PRAGMAs `journal_mode=WAL`, `synchronous=NORMAL`, `busy_timeout=5000`, `cache_size=-64000`, `foreign_keys=ON`, passive WAL checkpoint, then migrations (`packages/core/src/database/database.ts:22-55`).

### Legacy runtime — events published, core projector writes rows
- `Session.updateMessage` / `updatePart` publish `MessageUpdated` / `PartUpdated`; the core `SessionProjector` upserts `message` / `part` rows storing the v1 schema as JSON (`packages/core/src/session/projector.ts:260` onward). Deltas (`updatePartDelta`) are broadcast only (`packages/opencode/src/session/session.ts:877-885`).
- `EventV2Bridge` forwards durable events with `seq`/aggregate id as `sync` payloads (`packages/opencode/src/event-v2-bridge.ts:51-55`).

### v2 runtime — `EventV2` with in-transaction projectors
- Durable definitions declare `{aggregate: "sessionID", version}` (`packages/schema/src/session-event.ts:38-49`).
- `publish` → `commitDurableEvent` (`packages/core/src/event.ts:205-360`): inside `transaction(..., {behavior: "immediate"})` allocate the next per-aggregate `seq`, run all projectors registered for the type, run the optional local `commit(seq)` hook, insert the event row `{id, aggregate_id, seq, type: "<type>.<version>", data}` and update `event_sequence`; after commit wake durable tails.
- `commit` hook: "Local operational projection committed atomically with a new durable event. Not replayed or serialized" (`packages/core/src/event.ts:122-123`) — advances the Context Snapshot together with `ContextUpdated` ([[opencode--transcript-carried-system-prompt]]).
- Durable inventory (`packages/schema/src/session-event.ts:448-477`): AgentSwitched, ModelSwitched, Moved, Prompted, PromptAdmitted, ContextUpdated, Synthetic, Shell.*, Step.Started/Ended/Failed, Text.Started/Ended, Tool.Input.Started/Ended, Tool.Called/Progress/Success/Failed, Reasoning.Started/Ended, Retried, Compaction.Started/Ended, Revert.Staged/Cleared/Committed. Deltas are live-only (`packages/schema/src/session-event.ts:209`).
- Model context = SQL projection by `seq` ([[opencode--context-projection]]); pagination by `seq` because "Wall-clock timestamps may collide or move backwards" (`specs/v2/schema-changelog.md:378`).
- No durable execution identity: "Do not introduce an enclosing durable execution identity solely to group these facts; a process-local Session drain has no durable transcript boundary" (`specs/v2/todo.md:73-74`) → contrast [[durable-execution]].

## Constants
| name | value | path:line |
|---|---|---|
| `busy_timeout` | 5000 ms | `packages/core/src/database/database.ts:29` |
| `cache_size` | −64000 (64 MB) | `packages/core/src/database/database.ts:30` |
| event versions | session events v1; step settlement v2 | `packages/schema/src/session-event.ts:38-49` |

## Evolution
- 2026-02-13 `6d95f0d14c` SQLite storage.
- 2026-03-25 `b0017bf1b9` initial syncing (`event_sequence`).
- 2026-06-03 `76ee87ead8` v2 runtime; 2026-06-21 `fb43c15f88` (#33238) simplify event model.
- 2026-06-22 `reset_v2_session_state` migration.
- 2026-08-13 `d8bf79225f` `Moved` projector stops touching the transitional epoch table → [[projector-depends-on-transitional-table]].

## Quirks / drift
- Projectors carry a TODO: "Bind durable projectors to exact type+version before supporting incompatible historical payloads" (`packages/core/src/event.ts:180`).

Contrast: pi stores sessions as an append-only JSONL tree ([[pi--session-tree|pi]]) and pi-durable checkpoints task state; opencode makes every change an event whose projectors write SQLite rows in the same transaction.
