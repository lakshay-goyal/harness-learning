---
type: concept
stage: loop
tier: candidate
aliases: [runLoop, turn, turn_start, turn_end, agent-loop.ts, agent loop, inner loop, batch-early-termination, tool-presence-stop-promotion]
harnesses: [pi]
---
Inner agent loop: request → stream assistant → run its tool calls → append results → repeat while tool calls or injected messages exist; one iteration = one turn.

## Why
- Without an explicit loop contract, continuation is inferred from fragile signals (stop reason strings, tool presence) → truncated or half-finished responses get executed ([[length-truncated-tool-calls-executed]]).
- Re-entrant loops corrupt state when a second prompt starts mid-run ([[reentrant-prompt-corrupts-state]]); late callbacks leak events after a tool settles ([[late-tool-progress-after-settlement]]); listeners observing state before the reducer runs see stale data ([[listeners-see-stale-agent-state]]).
- Separating the stateless loop from session orchestration lets retry/compaction/queues live outside the hot path ([[run-settlement]]).

## Design space
- **Continue condition**: tool calls present (pi) · provider `stopReason == tool_use` · explicit model "done" tool. Tool-presence promotion must not override length/error stops (pi `5093641a5`).
- **Loop nesting**: single loop · inner turn loop + outer follow-up loop + session driver (pi: 3 levels).
- **Turn cap**: hard max iterations (common) · none, rely on user abort + bounded recovery (pi, [[no-turn-cap]]).
- **Tool batch execution**: parallel by default with sequential opt-out (pi) · sequential · model-chosen ([[parallel-tool-execution]]).
- **Early termination**: tool result hint `terminate` honored only when unanimous across the batch (pi) · any tool can end run · dedicated submit tool ([[structured-tool-output]]).
- **Failure exit**: error/aborted response = hard exit (pi) · auto-retry inside loop (pi moved retry outside, [[auto-retry-backoff]]).
- **Re-entrancy**: reject `prompt()` while running, force mid-run input through queues (pi) · implicit queueing.
- **State substrate**: in-memory messages array (pi stable) · chain of durable checkpointed tasks (pi-durable, [[durable-execution]]).

## Implementations
- [[pi--turn-loop|pi]] — stateless `runLoop` (inner turn + outer follow-up loop), parallel tools, length-stop guard, no turn cap; durable variant = generation/tool task chain.

## Failures
- [[reentrant-prompt-corrupts-state]]
- [[late-tool-progress-after-settlement]]
- [[listeners-see-stale-agent-state]]
- [[length-truncated-tool-calls-executed]]
- [[proxied-stream-option-loss]]

## Related
[[run-settlement]] · [[steering-queue]] · [[follow-up-queue]] · [[turn-lifecycle-hooks]] · [[abort-propagation]] · [[truncated-tool-call-guard]] · [[parallel-tool-execution]] · [[tool-error-as-result]] · [[message-conversion-layer]] · [[agent-event-stream]] · [[no-turn-cap]]
