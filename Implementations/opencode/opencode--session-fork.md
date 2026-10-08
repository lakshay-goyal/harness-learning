---
type: implementation
harness: opencode
concept: session-fork
commit: ecc4916b5a
files: [packages/opencode/src/session/session.ts:161-169, packages/opencode/src/session/session.ts:691-732]
---
[[session-fork]] in [[opencode]].

## Mechanism

### Legacy runtime — copy rows into a new session
- `Session.fork({sessionID, messageID?})` (`packages/opencode/src/session/session.ts:691-732`): new session in the current directory, same `workspaceID`, cloned `metadata`, title "<title> (fork #N)" with N incremented on re-fork (`getForkedTitle`, `packages/opencode/src/session/session.ts:161-169`).
- Copies messages **strictly before** `messageID` (all if omitted or not found), each with a fresh ascending `MessageID`; parts get fresh `PartID`s.
- **Remaps**: assistant `parentID` through the old→new id map (`packages/opencode/src/session/session.ts:710-716`); compaction part `tail_start_id` likewise (`packages/opencode/src/session/session.ts:725-727`), so a compacted fork keeps its retained tail ([[opencode--compaction-cut-point]]).
- No lineage link: `parentID` on sessions is reserved for subagent children, so a fork is not linked to its source (inference from `fork` not setting it).
- Files on disk are not touched; workspace rollback is a separate mechanism ([[opencode--workspace-snapshots]]).

### v2 runtime
- No fork operation on the v2 session service at HEAD (grep of `packages/core/src/session.ts` finds none).

## Constants
| name | value | path:line |
|---|---|---|
| fork title pattern | `"<title> (fork #N)"` | `packages/opencode/src/session/session.ts:161-169` |

## Evolution
- 2026-01-09 `a5edf3a311` (#6445) forked sessions with compactions broken by missing parent-child references → remap `parentID`.
- 2026-04-29 `d71b827d8c` (#24898) remap compaction `tail_start_id` when forking → [[fork-boundary-loss]].

## Quirks / drift
- Copying assigns new time-ordered ids but keeps the original `time.created`, so ordering "latest" by id vs by time can disagree in forked/imported sessions ([[reordered-context-misidentifies-latest-turn]], inference).
- A fork gets a new session id → new cache key and affinity, so it starts cold ([[opencode--session-affinity-cache-routing]]).

Contrast: [[pi--session-fork|pi]] writes a new JSONL with the root→target path and a `parentSession` header, re-chaining around label nodes; opencode copies SQLite rows with id remapping and no lineage.
