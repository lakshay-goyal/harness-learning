---
type: implementation
harness: codex
concept: session-token-budget
commit: 622e9e3696
files: [codex-rs/core/src/rollout_budget.rs:18, codex-rs/core/src/rollout_budget.rs:48, codex-rs/core/src/rollout_budget.rs:69, codex-rs/core/src/agent/control/budget.rs:11, codex-rs/core/src/session/rollout_budget.rs:7, codex-rs/core/src/context/rollout_budget.rs:5, codex-rs/core/src/config/mod.rs:1320, codex-rs/features/src/lib.rs:1849]
---
[[session-token-budget]] in [[codex]].

## Mechanism
- **Ledger**: `RolloutBudget` = "Shared accounting and reminder state for one root-thread session tree" (`codex-rs/core/src/rollout_budget.rs:18-28`), owned by the agent controller ("Keeps shared rollout-budget accounting and reminder state behind the controller", `codex-rs/core/src/agent/control/budget.rs:1-2`) → shared by all threads of the tree ([[in-process-subagent-threads]]).
- **Charge** (`record_usage`, `codex-rs/core/src/rollout_budget.rs:48-67`): if the response carries `usage.codex_rollout_budget_units` use it (must be finite, non-negative, else `Fatal`); otherwise `output_tokens × sampling_token_weight + non_cached_input × prefill_token_weight`. Returns true once exhausted "including on later calls".
- **Exhaustion**: `LocalAgentControl::record_rollout_budget_usage` → `CodexErr::SessionBudgetExceeded` ("shared rollout token budget exhausted") (`codex-rs/core/src/agent/control/budget.rs:11-17`; `codex-rs/protocol/src/error.rs:95`); terminal for retry (`codex-rs/protocol/src/error.rs:400`); compaction usage also counted (`codex-rs/core/src/compact_remote_v2.rs:296-297`) and compaction aborts immediately on it (`codex-rs/core/src/compact.rs:335-337`); compaction model-fallback skipped for it (`codex-rs/core/src/compact_model_fallback.rs:19`). `record_usage` contract: "`SessionBudgetExceeded` means the usage was recorded and the shared budget is now exhausted" (`codex-rs/core/src/agent/api.rs:124-127`). `TurnAbortReason::BudgetLimited` is handled like Interrupted (`codex-rs/core/src/tasks/mod.rs:594`) and maps child status to Interrupted (`codex-rs/core/src/agent/status.rs:15`).
- **Soft boundary**: "there is no cross-thread `Op::Interrupt` fanout. An in-flight thread can finish its current response before it observes the exhausted ledger, but every thread aborts at its next usage-accounting boundary" (`dac588f413` body).
- **Reminders**: thresholds `reminder_at_remaining_tokens`; `pending_reminder` computes `remaining = floor(max(limit − used, 0))` and the count of crossed thresholds; delivered once per thread per context window (`codex-rs/core/src/rollout_budget.rs:69-93`); marked delivered only after history insertion — "cancellation before then should retry it" (`:95-110`). Loop records it at the top of each iteration (`codex-rs/core/src/session/turn.rs:452-457`; `codex-rs/core/src/session/rollout_budget.rs:7-30`) as developer fragment `<rollout_budget>` "You have {n} weighted tokens left in the shared session token budget." (`codex-rs/core/src/context/rollout_budget.rs:5-31`).
- **Config**: `RolloutBudgetConfig{limit_tokens, reminder_at_remaining_tokens, sampling_token_weight, prefill_token_weight}` (`codex-rs/core/src/config/mod.rs:1320-1325`); TOML `[features.rollout_budget]` with `limit_tokens ≥ 1`, weights ≥ 0 (`codex-rs/features/src/feature_configs.rs:402-414`).

## Constants
| name | value | path:line |
|---|---|---|
| feature `rollout_budget` | `Stage::UnderDevelopment`, default off | `codex-rs/features/src/lib.rs:1849-1852` |
| default limit / weights | none (must be configured) | `codex-rs/features/src/feature_configs.rs:402-414` |
| reminder cadence | once per crossed threshold per thread per context window | `codex-rs/core/src/rollout_budget.rs:69-93` |

## Evolution
- 2026-06-18 `32a696dbac` "[codex] rollout budget implementation (varlength 2/N) (#28494)" — "`AgentControl` will now be the area where 'rollout level' features & accounting will have to live".
- 2026-06-19 `dac588f413` "[codex] abort turns when rollout budgets expire (token budget 3/3) (#28707)" — originally propagated as `CodexErr::TurnAborted`.
- 2026-06-22 `bd5bd953fb` "[codex] configure rollout budget reminder thresholds (#29423)".
- 2026-08-03 `8b8fa7276f` "Use provider-reported rollout budget units (#36715)".

## Quirks
- Distinct from the per-context-window `TokenBudget` reminders ([[model-requested-context-reset]]) and per-goal budgets ([[persistent-goal-continuation]]) — three budget mechanisms coexist.

## Versus pi
- pi has no spend or turn bound ([[no-turn-cap]]); usage is only reported ([[pi--usage-cost-accounting]]).
