---
type: implementation
harness: codex
concept: session-fork
commit: 622e9e3696
files: [codex-rs/core/src/thread_rollout_truncation.rs:35, codex-rs/protocol/src/protocol.rs:3192, codex-rs/core/src/thread_rollout_truncation.rs:96, codex-rs/core/src/thread_manager.rs:2570, codex-rs/app-server/src/request_processors/thread_processor.rs:5143, codex-rs/core/src/context_manager/history.rs:755, codex-rs/core/src/context_manager/history.rs:1364, codex-rs/core/src/agent/control/spawn.rs:88, codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279]
---
[[session-fork]] in [[codex]].

## Mechanism
- **User fork** (`thread/fork`, CLI `codex fork`, `codex exec fork`, TUI `/fork`): copies source rollout items into `InitialHistory::Forked(items)` cut strictly before the nth user message, rollback markers applied first when counting (`codex-rs/core/src/thread_rollout_truncation.rs:35-94`, `codex-rs/core/src/thread_manager.rs:2570-2605`). New thread id, new rollout file.
- **Lineage**: session meta carries `forked_from_id` and `forked_from_ordinal_exclusive` (`codex-rs/protocol/src/protocol.rs:3192-3196`).
- **Mid-turn source**: if n is out of range while the source is running, cut before the active turn's start so "the fork omits the unfinished suffix entirely" (`codex-rs/core/src/thread_manager.rs:2586-2597`).
- **Fork by turn id** (app-server): `truncate_rollout_after_turn_id` / `truncate_rollout_before_turn_id` require an explicit persisted `TurnStarted` boundary (`codex-rs/core/src/thread_rollout_truncation.rs:96-160`, `codex-rs/app-server/src/request_processors/thread_processor.rs:5143-5147`).
- **In-place rewind** `thread/revert` (experimental, paginated threads): replaces durable history with the prefix before `beforeTurnId`, keeps the thread id, writes a NEW immutable rollout file with a distinct `RolloutId`; "local file changes are unaffected" (`4ef836f883`, `4343b2bdc4`).
- **Rollback boundary unit** = "instruction turns": real user messages and inter-agent instructions; local compaction-summary messages are not boundaries (`codex-rs/core/src/context_manager/history.rs:1364-1379`). Rollback also strips first-turn `ModelSwitchInstructions` / `PersistentModeState` developer content (`codex-rs/core/src/context_manager/history.rs:822-846`) and clears the context baseline if it trims a mixed initial-context developer message (`:983-995`).
- **Sub-agent spawn = fork** (`spawn_agent`, multi-agent V2): `fork_turns` absent ⇒ `"all"` (full-history fork); `"none"` ⇒ fresh child; legacy positive integers accepted but treated as `"all"`; V1 `fork_context` rejected in V2 (`codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279-302`).
  - Child inherits only system/developer/user messages plus assistant messages with phase PartialAnswer/FinalAnswer; parent reasoning, tool calls/outputs, web-search/image calls, compaction triggers dropped; `TokenUsageRecord` dropped ("Child threads inherit model context, not the parent's cumulative usage state"); TurnContext/WorldState kept only when context baselines can be preserved (cache prefix) (`codex-rs/core/src/agent/control/spawn.rs:88-132`).
  - Parent-specific developer content stripped item by item (role instructions, usage hints, multi-agent mode text, current-time reminders, Guardian denial prefix) before the child's own instructions are appended (`codex-rs/core/src/agent/control/spawn.rs:134-165`).
  - Parent flushed before snapshotting store history (`codex-rs/core/src/agent/control/spawn.rs:1069`); compacted checkpoints in parent history rewritten into the child's initial history (`:1230-1260`).
  - V1 full fork forbids `agent_type` ("Full-history forked agents inherit the parent agent type; omit agent_type, or spawn without a full-history fork.", `codex-rs/core/src/agent/child_config.rs` `reject_full_fork_agent_type_override`); V2 allows a role since `82b17bc724`.
  - Durability: copied history may be enqueued under `PersistContext::SubagentSpawn`, but the caller must await standard persistence before acknowledging the child (`codex-rs/thread-store/src/store.rs:76-78`).

## Constants
| name | value | path:line |
|---|---|---|
| `fork_turns` default (V2) | `"all"` | codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:291 |
| partial (last-N) forks | removed; integers ⇒ all | codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279-302 |

## Evolution
- 2026-01-06 `8b7ec31ba7` (#8454) `thread/rollback` markers → removed 2026-09-11 `3052bbcf8c` (#44915).
- 2026-04-03 `567d2603b8` "Sanitize forked child history" → [[forked-child-inherits-parent-tool-noise]].
- 2026-04-21 `15b8cde2a4` (#18873) default multi-agent V2 fork to all.
- 2026-05-27 `61cbf3574e` (#24751), 2026-08-06 `a17da5e6e4` (#37260) → [[fork-carries-startup-context]].
- 2026-08-06 `82b17bc724` (#37252) roles allowed on full-history forks.
- 2026-08-12 `4ef836f883` (#38127), 2026-08-13 `4343b2bdc4` (#38440) `thread/revert`.
- 2026-08-20 `663da53823` per-item developer-content sanitization in forks.
- 2026-10-06 `6221a217e2` (#51329) partial-history sub-agent forks removed → [[no-partial-history-fork]].
- Workspace undo: ghost-commit `/undo` 2025-09-23 → 2025-12-22 un-shipped → [[undo-clobbers-user-git-state]], [[no-checkpoints-undo]].

## Quirks
- Fork is the default sub-agent context mode, chosen for cache-prefix reuse over context minimization (`6221a217e2`).
- Harness-injected context tied to removed turns must be pruned with the turns.

## Versus pi
- [[pi--session-fork]]: pi copies a root→leaf path into a new file with `parentSession` lineage and re-chains labels; codex copies a truncated linear prefix and records lineage in session meta `forked_from_id` + `forked_from_ordinal_exclusive` (`codex-rs/protocol/src/protocol.rs:3192-3196`), and reuses the same primitive for sub-agent spawn, which pi does not.
- pi offers in-file branching (`/tree`); codex has no in-file branch — revert writes a new file.
