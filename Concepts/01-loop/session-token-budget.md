---
type: concept
stage: cost
tier: variant
aliases: [RolloutBudget, RolloutBudgetConfig, RolloutBudgetReminder, RolloutBudgetContext, SessionBudgetExceeded, record_rollout_budget_usage, rollout budget, "<rollout_budget>", "TurnAbortReason::BudgetLimited"]
harnesses: [codex]
---
A weighted token budget shared by a whole root-thread agent tree (output tokens × sampling weight + non-cached input × prefill weight); crossing configured thresholds injects remaining-budget reminders into each thread, and exhausting it fails that thread's next usage update with a budget-exceeded error.

## Why
- Without a turn cap ([[no-turn-cap]]) and with sub-agents and auto-continuation, nothing bounds total spend of an autonomous run; a shared ledger bounds the whole tree.
- The model plans better when it knows the remaining budget; reminders must reach every thread, not just the one that crossed the threshold.
- Weighting reflects cost: cached input is cheap, output expensive.

## Design space
- **Scope**: per request · per thread · per root-thread session tree incl. sub-agents and compaction calls (✔ codex).
- **Units**: raw tokens · weighted (sampling × output + prefill × non-cached input) (✔ codex) · server-reported budget units override (✔ codex `usage.codex_rollout_budget_units`).
- **Enforcement**: hard cross-thread interrupt fan-out · soft: each thread fails at its next usage-accounting boundary (✔ codex; in-flight response may finish).
- **Reminders**: none · threshold list, delivered once per thread per context window, acknowledged only after history insertion (✔ codex).
- **Relation to per-goal budgets**: separate per-objective budget ([[persistent-goal-continuation]]) · per-context-window budget reminders ([[model-requested-context-reset]]).
- **None**: pi has no spend limit ([[no-turn-cap]]).

## Implementations
- [[codex--session-token-budget|codex]] — `RolloutBudget` ledger in `LocalAgentControl`; `RolloutBudgetContext` developer reminders; `CodexErr::SessionBudgetExceeded` terminal; feature `rollout_budget` under development.

## Failures
- [[goal-continuation-runaway]] (same class: unbounded autonomous spend)

## Related
[[no-turn-cap]] · [[in-process-subagent-threads]] · [[persistent-goal-continuation]] · [[usage-cost-accounting]] · [[auto-compaction]] · [[auto-retry-backoff]] · [[model-requested-context-reset]]
