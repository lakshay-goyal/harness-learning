---
type: implementation
harness: codex
concept: task-owned-subagent
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/multi_agents_spec.rs:692, codex-rs/core/src/tools/handlers/multi_agents_spec.rs:738, codex-rs/core/src/tools/handlers/multi_agents_spec.rs:772, codex-rs/core/src/tools/handlers/multi_agents_spec.rs:306, codex-rs/prompts/src/model_messages/multi_agent.rs:46, codex-rs/core/src/session/multi_agents.rs:93, codex-rs/core/src/agent/child_config.rs:20, codex-rs/core/src/config/mod.rs:259]
---
[[task-owned-subagent]] in [[codex]].

## Mechanism (ownership & lifecycle)
- Not task-owned in pi-durable's sense: children outlive the spawning tool call; the parent keeps working and is notified later. Ownership is by **agent tree** (hierarchical `AgentPath`, child = parent.join(task_name)), not by the tool task → mechanism in [[codex--in-process-subagent-threads|in-process-subagent-threads]]; results via [[codex--subagent-result-mailbox|subagent-result-mailbox]].
- Cancellation flows down by tree: V1 `close_agent` closes "an agent and any open descendants" (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:347`); V2 `interrupt_agent` rejects root/self, unloaded/dead target counts as interrupted (`codex-rs/core/src/agent/control/interrupt.rs:1-4`); suspension only when the root turn has no live descendants (`codex-rs/core/src/session/turn_suspension.rs:1-119`).
- Usage: descendants charged to the root goal and to the tree-wide rollout budget ([[persistent-goal-continuation]], [[session-token-budget]]).

## Mechanism (delegation policy in the tool description)
- **spawn_agent V1** description is a full delegation guide (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:692-770`): "Do not spawn sub-agents unless the user or applicable AGENTS.md/skill instructions explicitly ask for sub-agents, delegation, or parallel agent work. Requests for depth, thoroughness, research, investigation, or detailed codebase analysis do not count as permission to spawn." (`:738-739`); sections "When to delegate vs. do the subtask yourself" (critical path vs sidecar), "Designing delegated subtasks" ("each delegated task has a disjoint write set"), "After you delegate" ("Call wait_agent very sparingly", "Do not repeatedly wait by reflex"), "Parallel delegation patterns".
- **Model inheritance**: "Do not set the `model` field unless the user explicitly asks for a different model"; model list framed "Available model overrides (optional; inherited parent model is preferred)"; at most `MAX_SPAWN_AGENT_MODEL_OVERRIDES` = 5 listed (`codex-rs/core/src/agent/child_config.rs:20`) → [[subagent-config-inheritance]].
- **spawn_agent V2** (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:772-820`): hierarchical canonical task names (`/root/task1/task_3`); "The spawned agent will have the same tools as you and the ability to spawn its own subagents" (`:803`); context tradeoff "passing `fork_turns="none"` will not pass any surrounding context … whereas `fork_turns="all"` will provide the subagent with all surrounding context" (`:808`); explicit no-spawn instruction placed last.
- **wait_agent V2**: "Wait for a mailbox update from any live agent, including queued messages and final-status notifications. The wait also ends early when new user input is steered into the active turn. Does not return the content; returns either a summary of which agents have updates…" (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:306`).
- **close_agent V1**: "Completed agents remain open and count toward the concurrency limit until closed." (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:347`).
- **Mode toggles as superseding developer messages**: `EXPLICIT_REQUEST_ONLY_MULTI_AGENT_MODE_TEXT` "Any earlier instruction enabling proactive multi-agent delegation no longer applies…" vs `PROACTIVE_MULTI_AGENT_MODE_TEXT` "Proactive multi-agent delegation is active. Any earlier developer instruction requiring an explicit user request before spawning sub-agents no longer applies…" (`codex-rs/prompts/src/model_messages/multi_agent.rs:46-47`); Proactive only when effort is `Ultra`, else ExplicitRequestOnly, catalog text may override (`codex-rs/core/src/session/multi_agents.rs:93-104`) — cache-friendly (appended, not a system-prompt edit).
- **Orphaned prompts**: `codex-rs/core/templates/agents/orchestrator.md` and `codex-rs/core/templates/collab/experimental_prompt.md` are not referenced by any Rust code since `e47045c806` (2026-02-16) / `05b960671d` (2026-01-15), yet edited later (`cfd97b36da` 2026-03-13) — orphaned prompt files, no "used-by" check.

## Constants
| name | value | path:line |
|---|---|---|
| listed spawn model overrides | 5 | `codex-rs/core/src/agent/child_config.rs:20` |
| wait floor (anti busy-poll) | `DEFAULT_MULTI_AGENT_V2_MIN_WAIT_TIMEOUT_MS` = 10 000 | `codex-rs/core/src/config/mod.rs:259` |
| proactive delegation trigger | reasoning effort `Ultra` | `codex-rs/core/src/session/multi_agents.rs:93-104` |

## Evolution
- 2025-10-29 `13e1d0362d` "Delegate review to codex instance (#5572)" — first sub-agent-like use ([[review-subagent]]).
- 2026-01-06 `1dd1355df3` "feat: agent controller (#8783)"; 2026-01-12 `86f81ca010` / `623707ab58` / `9659583559` "collab" spawn/wait/close tools.
- 2026-01-14 `3d322fa9d8` "feat: add collab prompt (#9208)" — `experimental_prompt.md` ("This feature must be used wisely… you must tell them that they are not alone in the environment").
- 2026-01-15 `05b960671d` "feat: add agent roles to collab tools (#9275)" — collab prompt include removed, orchestrator role prompt added; `393a5a0311` / `7905e99d03` / `3c28c85063` (2026-01-15..19) iterate it: "You do not solve the task yourself" → "You may perform lightweight actions … but all substantive work must be delegated"; "spawn a verifier agent" removed in favour of self-verification; "**Your job is not finished until the entire task is fully completed and verified.** … You must not return early."; "Do not fix yourself unless the fixes are very small."
- 2026-01-25 `8fea8f73d6` "half max number of sub-agents" (12 → 6).
- 2026-01-26 `375a5ef051` "attempt to reduce high cpu usage when using collab (#9776)" — min wait + "Do not busy-poll `wait` with very short timeouts" → [[orchestrator-busy-polls-subagents]].
- 2026-02-16 `e41536944e` / `beb5cb4f48` renamed collab → multi_agent; `e47045c806` "feat: add customizable roles for multi-agents (#11917)" — orchestrator include removed ([[agent-roles]]).
- 2026-02-24 `dcab40123f` agent jobs `spawn_agents_on_csv`; removed 2026-07-20 `687f05cb94` ([[no-csv-fanout-jobs]]).
- 2026-03-04 `932ff28183` "feat: better multi-agent prompt (#13404)" — guide moved into `spawn_agent` description, incl. "A mini model can solve many tasks faster than the main model."
- 2026-03-11 `8f8a0f55ce` "spawn prompt (#14362)", `367a8a2210` "Clarify spawn agent authorization (#14432)" — explicit-permission rule + "Agent-role guidance … never authorizes spawning by itself" (regression test asserts strings) → [[over-eager-delegation]].
- 2026-03-24 `773fbf56a4` V2 communication pattern.
- 2026-04-20 `54bd07d28c` "[codex] prefer inherited spawn agent model (#18701)" — removed the mini-model sentence (test asserts absence) → [[subagent-model-downgrade]].
- 2026-06-12 `84520225b9` / 2026-06-15 `127224cacc` MAv2 prompts, `fork_turns` tradeoff text, no-spawn instruction last.
- 2026-09-21 `abbdde95b5` agent message board ([[agent-message-board]]).

## Quirks
- Behavioural text is placed in the `spawn_agent` description (not the system prompt) because the tool is not always present — placement follows "is this tool always present?" (M5b observation; contrast `update_plan`, [[tool-description-design]]).

## Versus pi
- [[pi--task-owned-subagent]]: pi-durable's child is owned by the tool task under structured concurrency (owner can't finish before owned work drains, bottom-up abort); codex children are owned by the agent tree and outlive the call, with asynchronous mailbox results and LRU capacity management.
- pi stable has no built-in subagents ([[no-subagents-core]]); codex multi-agent is a Stable default-on feature (`codex-rs/features/src/lib.rs:1422-1427`).
