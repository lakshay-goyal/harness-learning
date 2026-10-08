---
type: implementation
harness: pi
concept: session-fork
commit: b30a6dd77
files: [packages/coding-agent/src/core/agent-session-runtime.ts:262, packages/coding-agent/src/core/session-manager.ts:1632, packages/coding-agent/src/core/session-manager.ts:1815, packages/coding-agent/docs/sessions.md:20, packages/coding-agent/examples/extensions/git-checkpoint.ts:1, packages/durable/README.md:405]
---
[[session-fork]] in [[pi]].

## Mechanism
- **Three operations** (`packages/coding-agent/docs/sessions.md:24-30`): `/tree` moves within the file ([[session-tree]]); `/fork` "creates a new session from an earlier user message"; `/clone` "copies the active branch into a new session" (split from `/fork` UX in `d554409b1`, 2026-04-20).
- **`AgentSessionRuntime.fork(entryId, {position})`** (`packages/coding-agent/src/core/agent-session-runtime.ts:262-350`): emits `session_before_fork` (extensions may cancel, `:155-160,267-270`); `position:"before"` requires a user message and targets its parent, returning its text to the editor; `"at"` targets the entry itself (`:280-288`). Fork with no parent (first message) → fresh session with `parentSession` set (`:296-306`). Otherwise `createBranchedSession(target)`; tears down the current runtime (abort → `session_shutdown` → dispose) and builds a new one with `session_start{reason:"fork", previousSessionFile}` (`:299-345`).
- **`createBranchedSession(leafId)`** (`packages/coding-agent/src/core/session-manager.ts:1632-1725`):
  - path = `getBranch(leafId)` root→leaf.
  - **Labels dropped and re-chained**: label entries skipped, retained entries re-parented to the previous retained entry (`pathParentId`), so children of labels aren't orphaned (`:1639-1668`; `adf567c1c` #5669).
  - **Compaction remap**: a compaction whose `firstKeptEntryId` pointed at a dropped label is remapped to the next real entry; self-retaining compaction preserved (`:1650-1660`; `2631b25c3` #8989/#8990).
  - New UUIDv7 id, new file `<ts>_<id>.jsonl`, header `version: 3`, `parentSession: <previous file>` (`:1670-1682`).
  - Labels on retained entries recreated as fresh label entries chained after the path (`:1684-1711`).
  - Write rule shared with `_persist`: write now only if the path already has a conversation (`_hasConversation`), else defer creation (`:1713-1724`; `2f64df1e5` #1672 duplicate headers, unified in `ff72faba2`).
  - In-memory (non-persisted) mode replaces the current session with path + labels (`:1728+`).
- **`SessionManager.forkFrom(sourcePath, targetCwd)`**: copies a whole session file into another cwd's session dir with `parentSession` = source (`:1815-1866`); rejects empty/headerless sources.
- **Workspace state not forked**: conversation only; nothing restores files ([[no-checkpoints-undo]]). Example `git-checkpoint.ts`: `git stash create` at every `turn_start`, keyed by the leaf id seen at the last `tool_result` (`:14-27`); on `session_before_fork` offers `git stash apply <ref>` (UI only) (`:29-47`); checkpoints cleared on `agent_settled` (`:49-52`).
- **Confirmations**: `confirm-destructive.ts` example confirms clear/switch/fork via `before_*` events.

## Constants
| name | value | path:line |
|---|---|---|
| fork default position | `"before"` | packages/coding-agent/src/core/agent-session-runtime.ts:266 |
| forked header version | `CURRENT_SESSION_VERSION` (3) | packages/coding-agent/src/core/session-manager.ts:1675 |

## Evolution
- 2025-11-14 `8ae236f95` `/branch` command (copy to new session).
- 2025-12-25/26 `c58d5f20a`, `6f94e2462` session tree + branching API; `/fork` from user messages.
- 2026-01-16 `48b432415` session id resolution with fork support (#785).
- 2026-02-04 `13ac63c3c` fork writes to new file, not parent (#1242).
- 2026-02-27 `2f64df1e5` no duplicate headers when forking from pre-assistant entry (#1672).
- 2026-04-20 `d554409b1` `/clone` split from `/fork`.
- 2026-06-12 `adf567c1c` re-chain fork paths without labels (#5669).
- 2026-09-03 `2631b25c3` preserve compaction boundary when forking (#8990).
- 2026-09-25 `ff72faba2` shared `_hasConversation` write rule.
- #8937 in-memory fork before turn settled (06-fixes; hash unverified).

## Evidence commits
`8ae236f95`, `c58d5f20a`, `48b432415`, `13ac63c3c`, `2f64df1e5`, `d554409b1`, `adf567c1c`, `2631b25c3`, `ff72faba2`

## Quirks
- `git-checkpoint.ts` clears its map on `agent_settled`, but `/fork` is normally invoked when idle → the restore offer can only find checkpoints from the current run (inferred from code); `git stash create` also ignores untracked files.
- Fork drops abandoned branches but keeps `branch_summary` entries on the path (summaries reference `fromId` leaves that no longer exist in the new file — inferred).

## Durable variant (packages/durable)
- `conversation.fork(entryId, {ownership}, ctx)`: fork "sees its parent's entries up to `entryId` and continues independently" — no copy; inherits parent agent config as of that entry; **fresh provider session identity** (cache affinity) (`packages/durable/README.md:113,405-412`; `70eceaade`). Documents declare `fork: "initial"|"current"|"asOf"` (`packages/durable/README.md:516`). Storage conformance: deep fork history scans newest-first through ancestor caps; fork point exclusive of later entries in the same commit (`src/testing/storage-conformance.ts:367+`). Experimental services: "Durable branches by forking"; tree navigation dropped (`packages/coding-agent/src/experimental/services/README.md:38-45`).

## Failures
- [[fork-boundary-loss]] · [[fork-writes-wrong-file]]
