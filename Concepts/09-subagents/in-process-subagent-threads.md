---
type: concept
stage: subagents
tier: candidate
aliases: [multi-agent, "Collab feature", MultiAgentV2, MAv1, MAv2, MultiAgentVersion, AgentControl, LocalAgentControl, AgentRegistry, ThreadSpawn, "SubAgentSource::ThreadSpawn", AgentPath, task_name, "collaboration tool namespace", agent-graph-store, full-history fork]
harnesses: [codex]
---
Sub-agents are additional conversation threads hosted in the same process by a shared thread manager, addressed by a hierarchical task path, driven through model tools (spawn / message / follow-up / wait / interrupt / list), and returning their final answer as an injected message into the parent's mailbox.

## Why
- In-process threads share auth, model clients, exec policy, thread store and cache keys with the parent — no process spawn, and the harness sees every child's lifecycle, usage and approvals.
- Children that outlive the spawning call let the parent keep working (true parallelism) — but then results need a mailbox ([[subagent-result-mailbox]]) and capacity management ([[subagent-concurrency-limits]]).
- Forked children need the parent's conversation, not its identity/tool noise ([[forked-child-inherits-parent-tool-noise]]); config must come from the live turn ([[subagent-config-not-inherited]]).

## Design space
- **Host**: child CLI process ([[subagent-as-subprocess]], pi) · task-owned durable child conversation ([[task-owned-subagent]], pi-durable) · thread in the same thread manager (✔ codex) · remote worker behind the same controller trait (codex trait designed for it, only local backend in-tree) · remote hosted environment ([[cloud-task-delegation]]).
- **Addressing**: opaque UUID (codex before `79ad7b247b`) · model-chosen `task_name` joined onto the parent's path, duplicates rejected (✔ codex V2).
- **Child context**: fresh, seeded by a prompt (pi) · full-history fork by default, sanitized (✔ codex V2 `fork_turns="all"`) · last-N-turns fork (codex, removed `6221a217e2`) ([[session-fork]]).
- **Tool surface**: flat ids (codex V1: spawn/send_input/resume/wait/close) · Responses-API namespace `collaboration` with spawn/send_message/followup_task/wait_agent/interrupt_agent/list_agents, model-only (not from code mode) (✔ codex V2).
- **Tool text ownership**: bundled · model catalog may override descriptions AND parameter schemas per model, with bundled fallback (✔ codex).
- **Lifecycle verb**: close (frees slot) · interrupt only; idle children unloaded by LRU and reloadable by name (✔ codex V2).
- **Shared accounting**: tree-wide token budget and goal charging across descendants ([[session-token-budget]], [[persistent-goal-continuation]]).
- **Persistence**: spawn-edge graph store (open/closed edges) used for resume (✔ codex).
- **Workspace**: shared cwd (✔ codex, prompt-level "not alone in the codebase") · per-child worktree ([[managed-worktrees]] exists for threads only).

## Implementations
- [[codex--in-process-subagent-threads|codex]] — `AgentControl` trait / `LocalAgentControl` over `ThreadManager`; V1 vs V2 per turn; path addressing; namespaced tools; full-history sanitized forks; agent-graph-store.

## Failures
- [[wait-misses-already-queued-result]]
- [[forked-child-inherits-parent-tool-noise]] (08-state)
- [[subagent-eviction-loses-mail]]
- [[depth-capped-agent-still-has-tools]]

## Related
[[task-owned-subagent]] · [[subagent-as-subprocess]] · [[subagent-result-mailbox]] · [[subagent-config-inheritance]] · [[agent-roles]] · [[subagent-concurrency-limits]] · [[agent-message-board]] · [[session-fork]] · [[delegated-authorization-provenance]] · [[session-token-budget]] · [[no-subagents-core]] · [[subagent-hosting]] · [[request-attribution-metadata]]
