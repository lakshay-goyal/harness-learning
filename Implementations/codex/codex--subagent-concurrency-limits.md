---
type: implementation
harness: codex
concept: subagent-concurrency-limits
commit: 622e9e3696
files: [codex-rs/core/src/config/mod.rs:257, codex-rs/core/src/config/mod.rs:267, codex-rs/core/src/config/mod.rs:1633, codex-rs/core/src/agent/registry.rs:73, codex-rs/core/src/tools/spec_plan.rs:719, codex-rs/core/src/agent/control/residency.rs:100, codex-rs/core/src/agent/control/execution.rs:1, codex-rs/core/src/session/multi_agents.rs:93]
---
[[subagent-concurrency-limits]] in [[codex]].

## Mechanism
- **V1 count**: total spawned threads per user session ≤ `agents.max_concurrent_threads_per_session` (alias `max_threads`), default `DEFAULT_AGENT_MAX_THREADS` = 6, lock-free CAS counter (`codex-rs/core/src/config/mod.rs:257`; `codex-rs/core/src/agent/registry.rs:89-110`, `:333-349`); `0` is a config error (`codex-rs/core/src/config/mod.rs:3906-3915`).
- **V1 depth**: depth = parent depth + 1, rejected when > `agents.max_depth`, default `DEFAULT_AGENT_MAX_DEPTH` = 1 (only root may spawn), model-visible error "Agent depth limit reached. Solve the task yourself." (`codex-rs/core/src/config/mod.rs:267`; `codex-rs/core/src/agent/registry.rs:73-87`; `codex-rs/core/src/tools/handlers/multi_agents/spawn.rs:73-79`). At max depth V1 tools are not registered (`codex-rs/core/src/tools/spec_plan.rs:719-731`) → [[depth-capped-agent-still-has-tools]].
- **V2**: ignores `max_depth` ("Multi-agent v2 uses task-path routing and its own session/thread limits", `70ac0f123c`); children get tools only if the catalog marks the model V2-capable (`codex-rs/core/src/tools/spec_plan.rs:726-729`) → [[no-subagent-depth-limit-v2]]. Concurrency `DEFAULT_MULTI_AGENT_V2_MAX_CONCURRENT_THREADS_PER_SESSION` = 4 including root → 3 children (`codex-rs/core/src/config/mod.rs:258`, `:1633-1646`, `saturating_sub(1)`).
- **V2 residency LRU**: loaded children are an LRU; when a slot is needed an idle resident is unloaded (persisted, reloadable on next message) before failing with `AgentLimitReached` (`codex-rs/core/src/agent/control/residency.rs:100-140`) — why `close_agent` became `interrupt_agent` in V2: "V2 agents remain reusable by task name, and residency/unloading owns capacity management" (`8d415050fc`). Recipients pinned until queued submissions are handled; teardown in a separate task (`acc20df49f`) → [[subagent-eviction-loses-mail]].
- **Execution limiter** (running turns) separate from residency; root and MAv1 turns unrestricted (`codex-rs/core/src/agent/control/execution.rs:1-60`).
- **Stale slots**: `close_agent` on an agent whose thread already died returned an error and leaked the slot → reaped (`e2551a5e36`).
- **Propensity**: V2 sessions get "Proactive" delegation guidance only when reasoning effort is `Ultra`; otherwise "ExplicitRequestOnly" (`codex-rs/core/src/session/multi_agents.rs:93-104`).

## Constants
| name | value | path:line |
|---|---|---|
| V1 max threads per session | 6 (was 12) | `codex-rs/core/src/config/mod.rs:257` |
| V1 max depth | 1 | `codex-rs/core/src/config/mod.rs:267` |
| V2 concurrent threads incl. root | 4 (→ 3 children) | `codex-rs/core/src/config/mod.rs:258`, `:1633-1646` |
| V2 depth cap | none | `codex-rs/core/src/tools/spec_plan.rs:726-729` |

## Evolution
- 2026-01-25 `8fea8f73d6` "half max number of sub-agents" (`Some(12)` → `Some(6)`).
- 2026-03-26 `352f37db03` "fix: max depth agent still has v2 tools".
- 2026-04-27 `f8c527e529` V2 thread cap moved into feature config.
- 2026-04-29 `70ac0f123c` "Make multi-agent v2 ignore agents.max_depth".
- 2026-05-28 `e2551a5e36` "Reap stale multi-agent slots".
- 2026-06-08 `4e803a017c` "feat: add v2 agent residency lru"; `8d415050fc` close → interrupt.
- 2026-09-22 `acc20df49f` "Prevent V2 agent eviction from racing with queued messages".

## Versus pi
- pi example subagent: `MAX_PARALLEL_TASKS` 8 / `MAX_CONCURRENCY` 4 per call, no depth bound ([[pi--subagent-as-subprocess]]). Codex caps per session (6 / 4 incl. root), with a depth cap only in V1 and LRU unloading in V2.
