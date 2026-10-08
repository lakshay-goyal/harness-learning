---
type: tradeoff
concepts: [task-owned-subagent, subagent-as-subprocess, agent-profiles, session-handoff, in-process-subagent-threads, subagent-result-mailbox, subagent-config-inheritance, subagent-concurrency-limits, review-subagent, delegate-session-runner, cloud-task-delegation]
harnesses: [pi, opencode]
---
# builtin-subagents-vs-none

**Axis**: does the model get a built-in tool to delegate work to a child agent with its own context?

| option | pi | opencode | evidence |
|---|---|---|---|
| None; the human orchestrates (tabs, tmux) | ✅ stable | ❌ | pi: "Context transfer between agents is generally poor … You're the orchestrator." (`e9935beb5`); HEAD still skips it (`README.md:19`) → [[no-subagents-core]] |
| Subagent as a separate CLI process (extension) | ✅ `examples/extensions/subagent/` (8 parallel, 4 concurrent, 50 KiB cap) | — | `subagent/index.ts:33-36,300` → [[subagent-as-subprocess]] |
| In-process child session via `task` tool | experimental durable only (`packages/coding-agent/src/experimental/durable/subagent.ts:24-55`) | ✅ default: `general`, `explore`; resumable `task_id`; result wrapped in `<task_result>` | opencode: `packages/opencode/src/tool/task.ts:83-345`; `packages/opencode/src/agent/agent.ts:182-218` → [[task-owned-subagent]] |
| Nesting limit | durable child removes its own subagent extension (`subagent.ts:40`) | `subagent_depth ?? 1` since `285d315b4e` 2026-07-15 (unbounded before) | `packages/opencode/src/tool/task.ts:106-115` → [[unbounded-subagent-nesting]] |
| Background subagents | durable examples only | experimental flag; push completion, no polling tool | `packages/opencode/src/tool/task.ts:58-61`; `dabf2dc013` → [[no-background-task-polling]] |
| Permission inheritance | n/a | child gets parent session denies + `todowrite`/`task` deny, not the parent agent's restrictions | `packages/opencode/src/agent/subagent-permissions.ts:4-26` |

**When each wins**
- **No subagents (pi)**: tasks where full context beats summarized context, and where the human wants observability of every step. No double-counted usage, no permission-inheritance bugs, no recursion.
- **Built-in subagents (opencode)**: wide read-only exploration (`explore` keeps the parent's window clean), parallel independent work, and cheaper models for scouting. Costs: context loss at the boundary, mode-escape bugs (plan → general), recursion that needs a depth cap, and async coordination (polling vs push).

Related: [[no-subagents-core]] · [[plan-mode-vs-none]] · [[agent-profiles]] · [[pi]] · [[opencode]] · [[Tradeoffs]]

## Also: [[codex]] (folded from `subagent-hosting`)
**Axis** — where a delegated agent runs and how its result returns: no built-in delegation (user orchestrates; extension spawns a headless CLI subprocess; experimental durable child owned by the spawning tool task) vs. built-in in-process agent trees whose children outlive the call and report through an asynchronous mailbox.

| dimension | pi | codex |
|---|---|---|
| In core? | No — "pi does not and will not support sub-agents as a built-in feature… You're the orchestrator." (`e9935beb5`; HEAD `README.md:19`) ([[no-subagents-core]]) | Yes — multi-agent Stable, default-on (`codex-rs/features/src/lib.rs:1422-1427`); V1 collab + V2 namespaced tools chosen per turn (`codex-rs/core/src/config/mod.rs:1604-1631`) |
| Host | Example extension spawns `pi --mode json -p --no-session` subprocesses ([[pi--subagent-as-subprocess]], `packages/coding-agent/examples/extensions/subagent/index.ts:249`); pi-durable child conversation owned by a tool task (`packages/durable/docs/spec.md:2089`, [[pi--task-owned-subagent]]) | `CodexThread` in the parent's `ThreadManager`, behind an `AgentControl` trait designed for remote backends (`codex-rs/core/src/agent/api.rs:1-5`, [[codex--in-process-subagent-threads]]); hosted alternative `codex cloud` ([[codex--cloud-task-delegation]]) |
| Ownership / lifetime | Subprocess dies with the tool call; durable child owned by the tool task, owner can't finish before owned work drains, bottom-up abort | Agent tree by `AgentPath`; children outlive the spawning call, reusable by task name; interrupt by path; LRU unload + reload ([[codex--subagent-concurrency-limits]]) |
| Child context | Fresh, seeded by the task prompt; role prompt appended | Full-history fork by default, sanitized (only system/developer/user + final answers; parent role text stripped) (`codex-rs/core/src/agent/control/spawn.rs:88-165`); `fork_turns="none"` for fresh |
| Result return | Blocking tool result = child's final text, 50 KiB per task (`packages/coding-agent/examples/extensions/subagent/index.ts:36`); durable background → follow-up message | Asynchronous queue-only mail, assistant-role `FINAL_ANSWER` envelope, untruncated; `wait_agent` reports activity only (`codex-rs/core/src/context/inter_agent_completion_message.rs:23-45`, [[codex--subagent-result-mailbox]]) |
| Config inheritance | Explicit model wins else inherit model + thinking; tools from agent frontmatter | Whole live-turn runtime (provider, effort, approvals, sandbox, env, exec policy, service tier) via one helper ([[codex--subagent-config-inheritance]]) |
| Limits | Per call 8 tasks / 4 concurrent; recursion unbounded (example) | V1 6 threads/session, depth 1; V2 4 threads incl. root, no depth cap ([[no-subagent-depth-limit-v2]]) |
| Presets | Markdown agents (scout/planner/reviewer/worker) with tools allowlist | TOML roles restricted to narrowing overrides; built-in explorer/worker ([[codex--agent-profiles]]) |
| Delegation policy | User/extension decides | Model decides under "explicit request only" rule in the `spawn_agent` description; proactive mode at Ultra effort ([[codex--task-owned-subagent]]) |
| Internal uses | Side LLM calls without tools (compaction, summaries) | Full child sessions for review, approval reviewer, memory consolidation ([[codex--delegate-session-runner]], [[codex--review-subagent]]) |
| Shared accounting | Subprocess usage reported back by the tool | Tree-wide rollout budget, goal charging of descendants ([[session-token-budget]]) |
| Workspace | Same cwd (subprocess inherits) | Same cwd; prompt-level "not alone in the codebase" ([[no-subagent-workspace-isolation]]) |

**When each wins**
- **pi (none in core / subprocess)**: when context transfer loss outweighs parallelism (pi's stated reason), when users want to orchestrate sessions themselves, and when process isolation is the simplest way to get context isolation and crash containment. Cost: every lifecycle concern (abort, accounting, config inheritance, output caps) is re-implemented per extension ([[subagent-output-truncated]], [[subagent-config-not-inherited]]).
- **pi-durable task-owned**: crash-resumable runtimes that need deterministic cancellation cascades and idempotent replay.
- **codex in-process trees**: long parallel work where the parent must keep going, children are reused by name, and the harness must govern approvals, budgets, cache sharing and config centrally. Cost: a large failure surface around asynchrony — lost wakeups, eviction races, busy polling, over-eager spawning, model downgrades ([[wait-misses-already-queued-result]], [[subagent-eviction-loses-mail]], [[model-polls-background-work]], [[over-eager-delegation]], [[subagent-model-downgrade]]).
