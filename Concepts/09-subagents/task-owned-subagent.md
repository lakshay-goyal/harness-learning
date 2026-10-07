---
type: concept
stage: subagents
tier: candidate
aliases: [durable subagent tool, "ownership: {kind: task}", completing hold, structured-concurrency-ownership, background task, failFast, allSettled, keyed-service-generations, research task, scanConversations ownerTaskId, collab, multi_agent, spawn_agent, send_input, send_message, followup_task, wait_agent, close_agent, interrupt_agent, list_agents, resume_agent, orchestrator role, EXPLICIT_REQUEST_ONLY, PROACTIVE multi-agent mode, agent jobs, spawn_agents_on_csv]
harnesses: [pi, codex]
---
A subagent is a child conversation owned by the tool task that spawned it, under structured concurrency: abort flows down, the owner cannot finish before owned work drains, background tasks are explicit boundaries.

## Why
- Subprocess subagents are invisible to the harness's lifecycle: aborts, idle, crash recovery and accounting must be reimplemented per tool.
- Without an ownership tree, cancellations race (children outlive or miss parent aborts) ([[ownership-cancellation-races]]).
- Crash-resumable runtimes need idempotent spawn/submit so a replayed tool call finds the same child, and must not replay non-idempotent control operations ([[non-idempotent-tool-replayed-after-crash]]).
- Usage accounting double counts if the tool also reports the child's spend.
- The spawn tool's description is the delegation policy: explaining *how* to delegate reads as encouragement ([[over-eager-delegation]]), a model menu invites downgrades ([[subagent-model-downgrade]]), and wait tools get busy-polled ([[orchestrator-busy-polls-subagents]]).

## Design space
- **Ownership**: task-owned child conversation (pi-durable) · ownerless + manual bookkeeping · subprocess ([[subagent-as-subprocess]]) · owned by an agent tree (hierarchical task path), children outlive the spawning tool call (✔ codex, [[in-process-subagent-threads]]).
- **Foreground vs background**: tool waits for child answer (holds the run) · background anchor task that reports back as a follow-up message (pi-durable vacation / example 23) · always background: spawn returns `{task_name}` immediately, result arrives as queue-only mail, optional blocking `wait_agent` (✔ codex, [[subagent-result-mailbox]]).
- **Authorization to delegate**: model decides freely · only on explicit user/AGENTS.md/skill request, near-miss phrasings enumerated (✔ codex default) · proactive mode toggled by superseding developer message (✔ codex, Ultra effort).
- **Child model**: menu of cheaper models · inherit parent by default, overrides optional (✔ codex after `54bd07d28c`).
- **Batch fan-out**: CSV-driven job fan-out (codex `spawn_agents_on_csv`, removed `687f05cb94`) · per-call spawn.
- **Idempotence**: ownership index lookup + per-task `requestId` on submissions → `replay:"safe"` · not replay-safe for stop/kill.
- **Join policy**: `failFast` (abort siblings on first failure) · `allSettled`.
- **Abort order**: bottom-up so compensating handlers see final child outcomes.
- **Recursion**: child config removes the subagent extension · depth cap (codex V1 `max_depth` 1) · no depth cap, bounded by thread cap and model catalog (codex V2) ([[subagent-concurrency-limits]]).
- **Interrogation**: child conversation survives the call so users can switch to it.
- **Exposure to remote UIs**: keyed service instances per conversation with generation fencing (planned).

## Implementations
- [[pi--task-owned-subagent|pi]] — pi-durable ownership tree (Package 18/19) + experimental TUI `subagent` tool and vacation `research` background subagent.
- [[codex--task-owned-subagent|codex]] — built-in multi-agent tools (V1 collab / V2 namespaced) with agent-tree ownership; delegation policy lives in the `spawn_agent` description (explicit-request-only, inherit model, wait sparingly).

## Failures
- [[ownership-cancellation-races]]
- [[non-idempotent-tool-replayed-after-crash]]
- [[over-eager-delegation]]
- [[subagent-model-downgrade]]
- [[orchestrator-busy-polls-subagents]]
- [[wait-misses-already-queued-result]]

## Related
[[subagent-as-subprocess]] · [[durable-execution]] · [[crash-safe-tool-replay]] · [[abort-propagation]] · [[follow-up-queue]] · [[run-settlement]] · [[client-server-session-split]] · [[replicated-state]] · [[no-subagents-core]] · [[in-process-subagent-threads]] · [[subagent-result-mailbox]] · [[agent-roles]] · [[subagent-hosting]] · [[tool-description-design]]
