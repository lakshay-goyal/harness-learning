---
type: implementation
harness: codex
concept: session-tree
commit: 622e9e3696
files: [codex-rs/rollout/src/lib.rs:86, codex-rs/rollout/src/rollout_file_name.rs:39, codex-rs/history/src/lib.rs:210, codex-rs/history/src/lib.rs:357, codex-rs/rollout/src/policy.rs:16, codex-rs/rollout/src/policy.rs:78, codex-rs/rollout/src/recorder.rs:983, codex-rs/rollout/src/writer_lock.rs:17, codex-rs/rollout/src/compression.rs:364, codex-rs/thread-store/src/store.rs:68, codex-rs/protocol/src/protocol.rs:3348]
---
[[session-tree]] in [[codex]].

Codex variant: **linear** rollout JSONL per thread (no id/parentId tree, no leaf pointer); rewinds are replay markers (old) or a new rollout file per revert (now); forks are new threads ([[codex--session-fork|session-fork]]); a SQLite mirror indexes the files ([[codex--sqlite-session-index|sqlite-session-index]]).

## Mechanism
- **Location**: `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-<YYYY-MM-DDTHH-MM-SS>-<uuid>.jsonl`; archived threads move under `archived_sessions` (`codex-rs/rollout/src/lib.rs:86-87`, `codex-rs/rollout/src/rollout_file_name.rs:39-66`). CLI `codex archive|unarchive|delete`, TUI `/archive`, `/delete`, `/rollout` ("print the rollout file path", `codex-rs/tui/src/slash_command.rs:156`).
- **Line shape**: `RolloutLine {timestamp, ordinal?, type, payload}` with flattened `RolloutItem`; JSONL readers must use the canonical parser "so nested decimal values survive the flattened envelope" (`codex-rs/history/src/lib.rs:357-367`).
- **Item kinds**: SessionMeta, ResponseItem (with harness-metadata envelope), InterAgentCommunication(+Metadata), Compacted, TurnContext, TokenUsageRecord, WorldState, SecurityRiskScore, RetainedContext, EventMsg, RealtimeItem (`codex-rs/history/src/lib.rs:210-226`).
- **Persistence policy** (`codex-rs/rollout/src/policy.rs`):
  - every model-visible ResponseItem except `CompactionTrigger`/`Other` (`:78-101`);
  - `TurnContextItem` (cwd, approval/sandbox/permission profile, model, `comp_hash`, collaboration mode, effort…) once per real user turn and again after mid-turn compaction "so resume/fork replay can recover the latest durable baseline" (`codex-rs/protocol/src/protocol.rs:3348-3393`);
  - lifecycle events TokenCount, TurnStarted/Complete/Aborted, ThreadRolledBack, ThreadGoalUpdated, ThreadSettingsApplied;
  - **history mode**: `Legacy` persists UI events (UserMessage, AgentMessage, reasoning, PatchApplyEnd, McpToolCallEnd…); `Paginated` persists ItemCompleted TurnItems instead (`:128-155`);
  - transient events (exec deltas, approvals, stream errors, warnings, TurnDiff) never persisted (`:156-215`).
- **Persistence-only caps**: MCP results and command output in persisted items capped at 64 KiB with marker `"\n... command output truncated for persistence ...\n"` (`codex-rs/rollout/src/policy.rs:16-19`, `:250-275`). Function-call outputs inside ResponseItems are stored untruncated — live-history truncation only (`codex-rs/core/src/context_manager/history.rs:514-515`) → [[unbounded-payload-in-transcript]].
- **Compaction checkpoint** `CompactedItem`: summary `message`, full `replacement_history`, retained/guardian context, window number + ids, compaction response id, latest token-usage record ("thread/resume can restore token usage totals from this field without scanning arbitrarily far past the compaction"), resume metadata (`codex-rs/history/src/lib.rs:286-306`) → resume loads the newest compaction's replacement history and replays only the suffix ([[codex--context-projection|context-projection]]).
- **World state** persisted as `WorldStateItem{full, state}` with RFC 7386 merge patches (`codex-rs/protocol/src/protocol.rs:3330-3347`) → [[world-state-diff-injection]].
- **Deferred file creation**: new rollout buffered in memory (`deferred_creation: true`) until materialized; on failure "the recorder keeps all pending items in memory and a later persist() or flush() can retry" (`codex-rs/rollout/src/recorder.rs:983-992`, `:1061-1066`).
- **Single writer per thread**: lock files in `thread-writer-locks/` + `.coordination.lock` (`codex-rs/rollout/src/writer_lock.rs:17-18`).
- **Durability barriers** (`ThreadStore`, `PersistContext`): ThreadPreparation/Standard synchronous; SubagentSpawn/TurnStart/SteeredUserInput may be enqueued and fenced later (`codex-rs/thread-store/src/store.rs:68-94`) — accepted user input is durable before the next sampling request. Rollout flushed before `on_task_finished` and again after TurnAborted/TurnComplete (`codex-rs/core/src/tasks/mod.rs:357-374`, `:1023-1028`).
- **Cold compression**: rollouts ≥ 7 days compressed to `.jsonl.zst` by a best-effort background worker (max run 5 h; run marker / temp files stale after 6 h), read transparently (`codex-rs/rollout/src/compression.rs:30-33`, `:364-367`).
- **Torn tail**: trailing newline ensured before append on resume/compressed/direct paths (`e7d0e14172`) → [[torn-log-tail-fuses-next-entry]].
- **Debug trace** (separate from rollout): opt-in local trace bundle via `CODEX_ROLLOUT_TRACE_ROOT`, "observe first, interpret later" (`codex-rs/rollout-trace/README.md:1-20`).

## Constants
| name | value | path:line |
|---|---|---|
| `PERSISTED_MCP_RESULT_MAX_BYTES` | 64 KiB | codex-rs/rollout/src/policy.rs:16 |
| `PERSISTED_COMMAND_OUTPUT_MAX_BYTES` | 64 KiB | codex-rs/rollout/src/policy.rs:17 |
| `MIN_ROLLOUT_AGE` (compression) | 7 days | codex-rs/rollout/src/compression.rs:364 |
| `RUN_MARKER_STALE_AFTER` / `WORKER_MAX_RUNTIME` | 6 h / 5 h | codex-rs/rollout/src/compression.rs:365-367 |
| `WRITER_LOCK_DIR` | `thread-writer-locks` | codex-rs/rollout/src/writer_lock.rs:17 |
| session file size cap | none (function-call outputs uncapped) | codex-rs/core/src/context_manager/history.rs:514-515 |

## Evolution
- 2025-09-03 `234c0a0469` resume picker (`--resume` / `--continue`).
- 2026-01-06 `8b7ec31ba7` (#8454) `thread/rollback` appends `ThreadRolledBack{num_turns}` markers into the same JSONL; removed 2026-09-11 `3052bbcf8c` (#44915); markers still honored on replay (`codex-rs/core/src/context_manager/history.rs:755-880`).
- 2026-03-19 → 04-02 `2e03d8b4d2` rollout extracted into its own crate; 2026-04-14 `dae56994da` `ThreadStore` interface.
- 2026-04-30 `3516cb9751` (#20260) persisted MCP/command output capped at 64 KiB.
- 2026-06-24 `3e51b46eba` / `fa036d39aa` (#29833, #29835) world state persisted in rollouts.
- 2026-07-10 `e7d0e14172` (#32276) trailing newline before append.
- 2026-08-12 `4ef836f883` (#38127) / 2026-08-13 `4343b2bdc4` (#38440) `thread/revert` = new immutable rollout file with distinct `RolloutId` per revert.
- 2026-10-02 `bee28e8a06` paginated-history command output cap 64 KiB (subject only).

## Quirks
- The live in-memory `ContextManager` is the request source; the rollout is only read on resume/fork (`codex-rs/core/src/context_manager/history.rs:93`).
- Rewind and workspace state decoupled: "local file changes are unaffected" by `thread/revert` (`4343b2bdc4`) → [[no-checkpoints-undo]].
- Two history modes coexist; legacy rollouts are migrated by a crash-safe state machine → [[codex--session-migration|session-migration]].

## Versus pi
- [[pi--session-tree]]: one file holds an id/parentId tree + in-memory leaf; codex keeps the file linear and creates new files (fork, revert) instead of branches.
- pi writes per entry synchronously with no fsync; codex has explicit durability fences per `PersistContext` and a per-thread writer lock.
- Both repair torn tails by appending `\n` (pi `0b5ee5d8b`, codex `e7d0e14172`).
- codex adds persistence-only payload caps and cold `.zst` compression; pi has neither.
- See [[session-log-shape]].
