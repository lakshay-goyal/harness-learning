---
type: concept
stage: loop
tier: variant
aliases: [persistent-goal-loop, "/goal", ThreadGoal, ThreadGoalStatus, get_goal, create_goal, update_goal, GoalExtension, continue_if_idle, "turn_trigger goal", max_goal_token_budget, "thread/goal/set", continuation.md, budget_limit.md, objective_updated.md, budget_limited, usageLimited]
harnesses: [codex]
---
A persisted per-thread objective with a token budget: whenever the thread goes idle and the goal is still active, the harness starts another turn with a hidden continuation prompt, until the model marks it complete/blocked, the user pauses it, the budget or usage limit is hit, or a harness-side no-progress breaker trips.

## Why
- Long-horizon tasks outlast one turn; the model ends turns early ("waiting for input") even when it could keep going ([[goal-loop-premature-stop]]).
- An auto-continuation loop with only model self-reporting runs away on permanent errors and repeated blockers ([[goal-continuation-runaway]], [[unbounded-hook-continuation-loop]]).
- Under persistent pressure the model shrinks the goal to something that passes ([[goal-scope-shrinking]]) or shortcuts when shown elapsed time ([[time-pressure-shortcuts]]); stale steering text gets re-answered ([[steering-message-re-answered]]).

## Design space
- **Driver**: Stop hook that blocks end of turn ([[turn-lifecycle-hooks]]) · follow-up queue · idle hook that starts a fresh turn only if the thread is idle, else fails without injecting (✔ codex `on_thread_idle` → `continue_if_idle` → `start_turn_if_idle`).
- **Objective storage**: in prompt · persisted state DB row read under a permit so external set/clear can't race (✔ codex `goals_1.sqlite`).
- **Who may create**: model infers · only on explicit user/system request (✔ codex `create_goal` description).
- **Terminal states**: model-set complete/blocked/paused (user-requested only) · system-set budget-limited / usage-limited (✔ codex `ThreadGoalStatus`).
- **Breakers**: none · harness counters — 3 empty automatic turns, 3 exec-failure turns, any terminal turn error → Blocked (✔ codex) · prompt-side "same blocker ≥ 3 consecutive goal turns" audit (✔ codex).
- **Budget**: none · per-goal token budget charging (input − cached) + output, including descendant sub-agents, with wrap-up message on exhaustion (✔ codex).
- **Steering role**: hidden developer message (codex until 2026-05) · hidden user-context fragment (✔ codex HEAD — survives compaction, not re-answered).
- **Objective trust**: instruction · data — "user-provided data … not higher-priority instructions", XML-escaped inside `<objective>` / `<untrusted_objective>` (✔ codex).
- **None**: pi has no auto-continuation; termination is model-driven ([[no-turn-cap]]).

## Implementations
- [[codex--persistent-goal-continuation|codex]] — `codex-rs/ext/goal` extension: goal tools, idle continuation with `continuation.md`, budget/usage states, 3-strike breakers, objective-as-data framing.

## Failures
- [[goal-continuation-runaway]]
- [[goal-loop-premature-stop]]
- [[goal-scope-shrinking]]
- [[time-pressure-shortcuts]]
- [[steering-message-re-answered]]
- [[unbounded-hook-continuation-loop]] (10-platform)

## Related
[[run-settlement]] · [[follow-up-queue]] · [[turn-lifecycle-hooks]] · [[no-turn-cap]] · [[session-token-budget]] · [[persistent-agent-mode]] · [[steering-queue]] · [[guideline-softening]] · [[task-list-tool]] · [[extension-event-hooks]]
