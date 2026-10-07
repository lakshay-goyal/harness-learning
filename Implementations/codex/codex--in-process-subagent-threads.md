---
type: implementation
harness: codex
concept: in-process-subagent-threads
commit: 622e9e3696
files: [codex-rs/protocol/src/protocol.rs:3141, codex-rs/core/src/config/mod.rs:1604, codex-rs/core/src/tools/spec_plan.rs:1373, codex-rs/core/src/tools/spec_plan.rs:1473, codex-rs/core/src/tools/multi_agent_tool.rs:21, codex-rs/core/src/tools/handlers/multi_agents_common.rs:139, codex-rs/core/src/agent/api.rs:42, codex-rs/core/src/agent/control/spawn.rs:88, codex-rs/agent-graph-store/src/store.rs:22]
---
[[in-process-subagent-threads]] in [[codex]].

## Mechanism
1. **Host**: the child is a `CodexThread` in the same `ThreadManager` — no child process (`codex-rs/core/src/agent/control/spawn.rs`; `ThreadManager` owns threads, forks, subagent spawns, `codex-rs/core/src/thread_manager.rs:126`).
2. **Controller abstraction**: `codex-rs/core/src/agent/api.rs:1-5` "Coordination of one agent tree, independent of where its threads run … These Rust contracts do not define a wire protocol"; trait methods identity, resolve, spawn, send, take_mailbox, watch_mailbox, ensure_child_loaded, interrupt, list, child_agent_paths, check_turn_admission, admit_turn, record_usage, turn_finished, service_tier, propagate_config_update, get_guardian_package, pending_budget_reminder (`:42-160`). `AgentExecutionGuard`: "Remote backends must also recover reservations after worker loss" (`codex-rs/core/src/agent/types.rs:68-74`) — only `LocalAgentControl` exists in-tree (renamed `7fb599de8c`).
3. **Two generations, chosen per turn**: `MultiAgentVersion::{Disabled,V1,V2}` (`codex-rs/protocol/src/protocol.rs:3141`); `Feature::MultiAgentV2` forces V2, `agents.enabled=false` forces Disabled, else model catalog may pick, else `Feature::Collab` → V1 (`codex-rs/core/src/config/mod.rs:1604-1631`). Multi-agent is Stable, default-on (`codex-rs/features/src/lib.rs:1422-1427`).
4. **V1 tools** (flat ids): `spawn_agent`, `send_input`, `resume_agent`, `wait_agent`, `close_agent`; exposure `Deferred` (behind tool search) when the model supports search, else `Direct` (`codex-rs/core/src/tools/spec_plan.rs:1473-1500`, `:1476-1480`) → [[deferred-tool-loading]]. Not registered past max depth (`:719-731`).
5. **V2 tools** in a Responses-API namespace (default `collaboration`, `codex-rs/core/src/config/mod.rs:262`): `spawn_agent`, `send_message`, `followup_task` (both omitted when `disable_direct_message`), `wait_agent` (omitted when `wait_agent_enabled=false`), `interrupt_agent`, `list_agents` (`codex-rs/core/src/tools/spec_plan.rs:1373-1470`); default exposure `DirectModelOnly` — not callable from code mode (`:1381-1385`; default `codex-rs/core/src/config/mod.rs:1386`) → [[code-mode]]. Child sessions get V2 tools only if the catalog marks the model V2-capable (`codex-rs/core/src/tools/spec_plan.rs:726-729`).
6. **Catalog-owned tool text**: per-model `multi_agent_tool_description_override` / `multi_agent_tool_parameters_override`; invalid schemas fall back to bundled with a warning; parameters marked `encrypted` keep the marker so agent-to-agent content can travel encrypted (`codex-rs/core/src/tools/multi_agent_tool.rs:21-55`; `AgentMessage::{Plaintext,Encrypted}` `codex-rs/core/src/agent/types.rs:63-66`) → [[per-model-system-prompt]].
7. **Addressing**: V2 `task_name` joined onto the parent's `AgentPath` (`codex-rs/core/src/tools/handlers/multi_agents_common.rs:139-164`); duplicates rejected "agent path `…` already exists" (`codex-rs/core/src/agent/registry.rs:299-317`).
8. **Spawn result** is tiny: `{task_name}` (or `{task_name, nickname}` when `hide_spawn_agent_metadata=false`; default true since `668703c23f`) (`codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:250-260`, `:311-320`); `send_message`/`followup_task` return an empty string (`codex-rs/core/src/tools/handlers/multi_agents_v2/message_tool.rs`).
9. **Full-history fork by default** (V2): `fork_turns` absent ⇒ "all"; "none" ⇒ fresh; legacy positive integers accepted but mean "all"; V1 `fork_context` rejected in V2 (`codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279-302`) → [[no-partial-history-fork]]. Fork sanitization keeps only system/developer/user messages + assistant PartialAnswer/FinalAnswer; drops parent reasoning, tool calls/outputs, web-search/image calls, compaction triggers, `TokenUsageRecord` ("Child threads inherit model context, not the parent's cumulative usage state"); keeps TurnContext/WorldState only when baselines can be preserved (cache prefix) (`codex-rs/core/src/agent/control/spawn.rs:88-132`); strips parent-specific developer content (role instructions, usage hints, multi-agent mode text, current-time reminders, Guardian denial prefix) by kind (`:134-165`); parent flushed before snapshot (`:1069`); compacted checkpoints rewritten into the child's initial history (`:1230-1260`) → [[session-fork]], [[forked-child-inherits-parent-tool-noise]].
10. **Roles on forks**: V1 forbids `agent_type` on full-history forks ("Full-history forked agents inherit the parent agent type; omit agent_type, or spawn without a full-history fork.", `codex-rs/core/src/agent/child_config.rs` `reject_full_fork_agent_type_override`); V2 allows (`82b17bc724`) → [[agent-profiles]].
11. **Status derivation**: TurnStarted→Running, TurnComplete→Completed(last_agent_message) or Errored, TurnAborted(Interrupted|BudgetLimited)→Interrupted, others→Errored, ShutdownComplete→Shutdown; final = not PendingInit/Running/Interrupted (`codex-rs/core/src/agent/status.rs`).
12. **Graph persistence**: `codex-rs/agent-graph-store` — storage-neutral parent/child spawn-edge store with Open/Closed status and status-filtered descendant walks (`codex-rs/agent-graph-store/src/store.rs:22-60`, `codex-rs/agent-graph-store/src/types.rs:7-11`); resume = BFS over children with depth+1 (`codex-rs/core/src/agent/control/spawn.rs:1345-1415`).
13. **list / interrupt**: `list_agents` returns name+status only (`64c0e2fa1b`); V1 `close_agent` returns agent snapshots (`fdd89e78ac`).
14. **Workspace**: children share the parent's cwd (`config.cwd = turn_cwd`, `codex-rs/core/src/agent/child_config.rs`); mitigation is prompt-level (`worker` role "not alone in the codebase", `codex-rs/core/src/agent/role.rs:381`) → [[no-subagent-workspace-isolation]].
15. **Request attribution**: `x-openai-subagent: collab_spawn|review|compact|memory_consolidation|<label>` header (`codex-rs/codex-api/src/requests/headers.rs:16-31`); cache key moved from thread id to session id so root and children share cache shards (`4aa950d456`) → [[request-attribution-metadata]], [[session-affinity-cache-routing]].
16. **World state**: the parent's `<subagents>` section lists at most 8 children / 1 024 bytes (`codex-rs/core/src/session/world_state.rs:40-41`) → [[world-state-diff-injection]].

## Constants
| name | value | path:line |
|---|---|---|
| V2 tool namespace | `"collaboration"` | `codex-rs/core/src/config/mod.rs:262` |
| world-state subagent list cap | 8 entries / 1 024 bytes | `codex-rs/core/src/session/world_state.rs:40-41` |
| `hide_spawn_agent_metadata` default | true | `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:250-260` |

## Evolution
- 2026-01-06 `1dd1355df3` agent controller; 2026-01-12 `623707ab58` / `9659583559` wait/close tools.
- 2026-03-20 `79ad7b247b` "feat: change multi-agent to use path-like system instead of uuids (#15313)".
- 2026-03-23 `450dc289c3` V2 split; 2026-03-24 `38c088ba8d` `list_agents`; 2026-03-30 `213756c9ab` mailbox concept.
- 2026-04-03 `567d2603b8` "Sanitize forked child history"; 2026-04-21 `15b8cde2a4` "default multi-agent v2 fork to all (#18873)".
- 2026-04-29 `782191547c` agent graph store; reverted 2026-05-06 `a8488fec5e`; re-injected 2026-06-24 `ece1dfece0`.
- 2026-06-03 `668703c23f` hide spawn metadata by default; 2026-06-08 `8d415050fc` `close_agent` → `interrupt_agent` in V2.
- 2026-07-14 `64c0e2fa1b` `list_agents` without task messages; `4aa950d456` session-id cache keys.
- 2026-08-06 `82b17bc724` roles on full-history forks; 2026-08-20 `663da53823` developer-content sanitization in forks.
- 2026-09-18 `7fb599de8c` `LocalAgentControl` rename; 2026-09-22 `fdd89e78ac` close returns snapshots.
- 2026-10-06 `6221a217e2` "Remove partial-history subagent forks (#51329)".

## Versus pi
- pi: no core subagents ([[no-subagents-core]]); example extension spawns `pi --mode json -p --no-session` subprocesses with fresh context and returns final text ([[pi--subagent-as-subprocess]]); pi-durable owns children by tool task ([[pi--task-owned-subagent]]).
- codex: in-process threads sharing services, full-history fork by default, asynchronous mailbox results, path addressing, catalog-owned tool text. See [[builtin-subagents-vs-none]].
