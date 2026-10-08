---
type: concept
stage: loop
tier: must-have
aliases: [runLoop, turn, turn_start, turn_end, agent-loop.ts, agent loop, inner loop, batch-early-termination, tool-presence-stop-promotion, SessionPrompt.runLoop, Session Drain, Provider Turn, SessionRunner.run, needsContinuation, isOrphanedInterruptedTool, run_turn, run_sampling_request, try_run_sampling_request, SamplingRequestResult, needs_follow_up, end_turn, sampling request, StepContext, drain_in_flight]
harnesses: [pi, opencode, codex]
---
Inner agent loop: request → stream assistant → run its tool calls → append results → repeat while tool calls or injected messages exist; one iteration = one turn.

## Why
- Without an explicit loop contract, continuation is inferred from fragile signals (stop reason strings, tool presence) → truncated or half-finished responses get executed ([[length-truncated-tool-calls-executed]]).
- Re-entrant loops corrupt state when a second prompt starts mid-run ([[reentrant-prompt-corrupts-state]]); late callbacks leak events after a tool settles ([[late-tool-progress-after-settlement]]); listeners observing state before the reducer runs see stale data ([[listeners-see-stale-agent-state]]).
- Separating the stateless loop from session orchestration lets retry/compaction/queues live outside the hot path ([[run-settlement]]).
- Optional warm-up work placed before the loop becomes visible gates turn start and interrupt ([[prewarm-blocks-turn-start]]); "input is pending" as a continue condition turns terminal failures into retry loops ([[pending-input-restarts-failed-turn]]).

## Design space
- **Continue condition**: tool calls present (pi) · provider `stopReason == tool_use` · explicit model "done" tool · any dispatched tool call OR pending user/agent input OR server `end_turn == false` (✔ codex `needs_follow_up = model_needs_follow_up || has_pending_input`). Tool-presence promotion must not override length/error stops (pi `5093641a5`).
- **Loop nesting**: single loop · inner turn loop + outer follow-up loop + session driver (✔ pi: 3 levels) · task wrapper (`RegularTask` re-runs `run_turn` while input pending) → outer turn loop → per-request retry loop → stream loop (✔ codex: 4 levels).
- **Tool start time**: after the full assistant message (✔ pi) · as soon as each `output_item.done` arrives mid-stream, results awaited in model order after the stream (✔ codex `FuturesOrdered`).
- **Per-request re-capture**: tools/settings/context captured per user prompt · fresh per-step snapshot (tools, MCP binding, settings, AGENTS.md) before every sampling request (✔ codex `StepContext`; pi via `prepareNextTurn` hooks).
- **Turn cap**: hard max iterations (common) · none, rely on user abort + bounded recovery (✔ pi, ✔ codex — comment relies on compaction, [[no-turn-cap]]) · economic budgets ([[session-token-budget]], [[persistent-goal-continuation]] breakers, codex).
- **Tool batch execution**: parallel by default with sequential opt-out (pi) · sequential · model-chosen · per-tool reader/writer lock inside one batch (codex) ([[parallel-tool-execution]]).
- **Early termination**: tool result hint `terminate` honored only when unanimous across the batch (pi) · any tool can end run · dedicated submit tool ([[structured-tool-output]]) · Stop hook may block end and inject a continuation prompt (codex, [[turn-lifecycle-hooks]]).
- **Failure exit**: error/aborted response = hard exit (pi) · auto-retry inside loop (codex per sampling request; pi moved retry outside, [[auto-retry-backoff]]) · non-retryable error → error event, break, "let the user continue" (codex).
- **Re-entrancy**: reject `prompt()` while running, force mid-run input through queues (pi) · implicit queueing · one op that steers if a turn is active else starts one, plus single task slot (✔ codex, [[single-active-task-slot]]).
- **State substrate**: in-memory messages array (pi stable) · chain of durable checkpointed tasks (pi-durable, [[durable-execution]]) · append-only history + rollout flushes before terminal events, turn-granular suspend/recover (codex).
- **State substrate (store-driven)**: no in-memory transcript; every step re-reads history from SQLite and re-derives the newest user / assistant / finished assistant (opencode legacy and v2).
- **Continue condition (transcript-derived)**: continue while the last assistant has non-provider-executed tool parts or does not answer the newest user message, whatever the finish reason; `unknown` finish continues (opencode).
- **Loop nesting (durable inbox)**: inner provider-turn loop + outer queue-promotion loop per Session Drain (opencode v2).
- **Turn cap (opt-in wrap-up)**: per-agent step budget that forces a text-only final answer instead of erroring (opencode, [[step-budget-limit]]).

## Implementations
- [[pi--turn-loop|pi]] — stateless `runLoop` (inner turn + outer follow-up loop), parallel tools, length-stop guard, no turn cap; durable variant = generation/tool task chain.
- [[codex--turn-loop|codex]] — `run_turn` outer loop (pending-input drain, hooks, per-step `StepContext`, reminders, world-state diff) → `run_sampling_request` retry loop → `try_run_sampling_request` stream loop with tools started mid-stream; `needs_follow_up` continue rule; no turn cap.
- [[opencode--turn-loop|opencode]] — legacy `SessionPrompt.runLoop` re-reads SQLite each step, one AI SDK `streamText` per step; v2 Session Drain = inner provider-turn loop + outer queue promotion.

## Failures
- [[reentrant-prompt-corrupts-state]]
- [[late-tool-progress-after-settlement]]
- [[listeners-see-stale-agent-state]]
- [[length-truncated-tool-calls-executed]]
- [[proxied-stream-option-loss]]
- [[prewarm-blocks-turn-start]]
- [[pending-input-restarts-failed-turn]]
- [[stop-reason-mapping-gaps]] (02-model-interface)
- [[rewrite-introduces-unrequested-limits]]
- [[identical-tool-call-loop]]

## Related
[[run-settlement]] · [[steering-queue]] · [[follow-up-queue]] · [[turn-lifecycle-hooks]] · [[abort-propagation]] · [[truncated-tool-call-guard]] · [[parallel-tool-execution]] · [[tool-error-as-result]] · [[message-conversion-layer]] · [[agent-event-stream]] · [[no-turn-cap]] · [[single-active-task-slot]] · [[mid-turn-settings-switch]] · [[world-state-diff-injection]] · [[current-time-reminder]] · [[session-token-budget]]

## Tradeoffs
- [[turn-cap-vs-none]]
