---
type: concept
stage: subagents
tier: candidate
aliases: [max_concurrent_threads_per_session, max_threads, agents.max_depth, AgentLimitReached, AgentExecutionLimiter, V2Residency, residency LRU, spawn_guard, "Agent depth limit reached"]
harnesses: [codex]
---
Caps on how many child agents exist and run, how deep delegation may nest, and how capacity is reclaimed (close vs LRU unloading of idle children).

## Why
- Fan-out multiplies cost and load; models spawn more than needed ([[over-eager-delegation]]).
- Recursion without a bound explodes; but a depth cap that still exposes spawn tools produces deterministic failures ([[depth-capped-agent-still-has-tools]]).
- Completed children that hold slots until closed starve new work; unloading them must not lose their mail ([[subagent-eviction-loses-mail]]).

## Design space
- **Count**: per-call parallel/concurrency caps (pi example 8 / 4) · per-session total spawned threads (codex V1 6) · concurrent threads incl. root (codex V2 4 → 3 children).
- **Depth**: unbounded (pi example) · cap with model-visible error "Solve the task yourself" and tools hidden at max depth (codex V1, depth 1) · none, bounded by model catalog (which models get tools) + thread cap (codex V2, [[no-subagent-depth-limit-v2]]).
- **Capacity reclaim**: explicit close (codex V1 "Completed agents remain open and count toward the concurrency limit until closed") · LRU unload idle residents, reload on next message (codex V2).
- **Running vs resident**: one limit · separate execution limiter for running turns vs residency for loaded threads (✔ codex).
- **Delegation propensity**: proactive vs explicit-request-only mode by effort level (✔ codex).

## Implementations
- [[codex--subagent-concurrency-limits|codex]] — V1 `max_concurrent_threads_per_session` 6 + `max_depth` 1; V2 4 threads incl. root, residency LRU, execution limiter, no depth cap.

## Failures
- [[depth-capped-agent-still-has-tools]]
- [[subagent-eviction-loses-mail]]

## Related
[[in-process-subagent-threads]] · [[subagent-result-mailbox]] · [[task-owned-subagent]] · [[subagent-as-subprocess]] · [[session-token-budget]] · [[no-subagent-depth-limit-v2]] · [[subagent-hosting]]
