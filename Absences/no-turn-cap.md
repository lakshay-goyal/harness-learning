---
type: absence
harnesses: [pi, codex]
---
# no-turn-cap

**What's missing**
- No maximum turns, steps, iterations or wall-clock budget per prompt in the agent loop or the coding-agent session.
- `git grep -E 'maxTurns|maxIterations|maxSteps' -- 'packages/*/src'` → no hits at `b30a6dd77`.
- `packages/agent/src` contains **zero numeric literals ≥ 10** (`grep -rnE '[0-9]{2,}' packages/agent/src` → empty).
- The durable harness also has no turn cap (same grep over `packages/durable/src`).

**Evidence of decision**
- No explicit manifesto line. This is an absence by omission plus design: the loop is `while (true)` (`packages/agent/src/agent-loop.ts:179`) and ends only when:
  - the response has stopReason `error` / `aborted` (`:245-256`);
  - `finishTurn` returns `{action:"end"}` (`:289-292`);
  - there are no tool calls, steering, follow-ups or continuation (`:316-317`);
  - every tool result has `terminate: true` (`:272,690-692`);
  - the session is aborted (`packages/coding-agent/src/core/agent-session.ts:1832-1839`).
- The docs acknowledge the risk instead of capping it. `packages/agent/README.md:146`: "`finishTurn` runs again after the next request, so returning `{ action: \"continue\" }` unconditionally creates an endless loop."

**What bounds a run instead**
- The human: Esc → `session.abort()` ([[abort-propagation]]; `modes/interactive/interactive-mode.ts:3050-3052`).
- Retries: `retry.maxRetries = 3` with a 60s per-wait cap (`settings-manager.ts:1026`, `packages/ai/src/utils/retry.ts:126`). The cap was added in `c37b0e03b` (#8826) after backoff grew unbounded during an outage → [[auto-retry-backoff]].
- Overflow recovery runs **once** per user turn via `_overflowRecoveryAttempted`. Added in `6b4b92042` (#1319, "stop overflow auto-compaction cascades") → [[overflow-recovery]].
- Context window plus compaction ([[auto-compaction]]).

**Opt-in replacement**
- A `turn_end` / `finishTurn` hook can count turns and return `{action:"end"}` ([[turn-lifecycle-hooks]]). No shipped example caps turns (unverified beyond the examples index).
- `structured-output.ts` example: a `terminate: true` tool ends the run deterministically ([[structured-tool-output]]).

**History**
- No cap has ever existed: pickaxe for `maxTurns` finds nothing (`07-constants` census).

**Implication**
- pi trusts the model and the watching human. Unattended print/RPC/SDK runs can loop until context or money runs out. Embedders must add their own budget via hooks.
- Every self-healing path is bounded to one attempt (overflow) or to N with a cap (retry) instead. Bound the recovery loops, not the work loop.

**codex** — *absent too*. `run_turn`'s `loop` has no counter; exits are: no follow-up needed, hook stop/block-without-prompt, errors, abort (`codex-rs/core/src/session/turn.rs:424-837`); a comment relies on compaction to avoid infinite looping (`codex-rs/core/src/session/turn.rs:588`). No `max_turns` / `max_steps` / `max_iterations` / loop detection in `codex-rs/**/*.rs` (grep; only TUI display constant `RECAP_HISTORY_MAX_TURNS = 8`, `codex-rs/tui/src/app/recap_history.rs:13`). Bounds are economic / terminal states instead: shared rollout token budget → `SessionBudgetExceeded` / BudgetLimited (`codex-rs/core/src/agent/control/budget.rs:5-12`, `codex-rs/core/src/rollout_budget.rs:1-121`) → [[session-token-budget]]; usage-limit errors; goal `blocked` / `usageLimited` states (`0d344aca9b` 2026-05-18) and no-progress breakers (3 empty / 3 exec-failure / 3 blocked turns) → [[persistent-goal-continuation]]; token budgeting on by default for supporting models (`d58d0e5841` 2026-08-31). Stop-hook continuations uncapped → [[unbounded-hook-continuation-loop]].
**opencode contrast**: implements it, opt-in: per-agent `steps` (default `Infinity`) with a forced wrap-up message, plus a 3-identical-call guard (`packages/opencode/src/session/prompt.ts:1178`; `packages/opencode/src/session/processor.ts:29`) — see [[step-budget-limit]] / [[repeated-tool-call-detection]] / [[turn-cap-vs-none]].

Related: [[turn-loop]] · [[turn-lifecycle-hooks]] · [[abort-propagation]] · [[auto-retry-backoff]] · [[overflow-recovery]] · [[run-settlement]] · [[no-bash-default-timeout]] · [[Absences]] · [[session-token-budget]] · [[persistent-goal-continuation]] · [[unbounded-hook-continuation-loop]] · [[goal-continuation-runaway]]
