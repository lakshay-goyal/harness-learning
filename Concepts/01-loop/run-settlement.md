---
type: concept
stage: loop
tier: candidate
aliases: [agent_settled, agent_before_settle, _handlePostAgentRun, _runAgentPrompt, waitForIdle, willRetry, post-run driver, awaited listeners, settle boundary]
harnesses: [pi]
---
Explicit "no more automatic work" boundary, distinct from the end of one model loop; a post-run driver inspects the last response and decides retry / compaction / continue-for-queued-work / settle.

## Why
- One user prompt can span several low-level runs (retry, overflow compact-and-retry, queued follow-ups); callers that treat end-of-loop as done exit mid-retry or see stale idle state ([[retry-wait-race-prompt-returns-early]]).
- Queued work added by end-of-run handlers is stranded unless the driver re-checks queues after them ([[queued-messages-stranded-at-run-end]]).
- Fire-and-forget event handlers persist out of order; awaited listeners + a barrier make persistence part of settlement.
- "Idle" that ignores side operations (compaction, summaries) reports success while work continues ([[compaction-cancellation-races]]).

## Design space
- **Driver shape**: fire-and-forget handlers + promise/timer continuations (pi early, `setTimeout(100)`) · serialized event queue + retry promise (pi 2026-03) · synchronous post-run loop (pi HEAD, `32bcdc973`).
- **Settle signal**: explicit event (`agent_settled`, pi) · infer from end-of-loop + flags (`willRetry`) · polling idle.
- **Extension hook at settle**: before-settle boundary allowed one continuation (pi) · none.
- **Listener semantics**: awaited, ordered, part of idle (pi `9022a5b5e`) · observational only (pi raw `agentLoop()`).
- **Idle definition**: no run · no run AND no compaction/summary/retry (pi).
- **Durable**: run baton document (`pi.live.run`) whose presence = busy; inputs settle `done`/`unanswered` atomically (pi-durable).

## Implementations
- [[pi--run-settlement|pi]] — `_runAgentPrompt` driver loop: retry → compaction → queued → `agent_before_settle` → `agent_settled`; `agent_end.willRetry`.

## Failures
- [[retry-wait-race-prompt-returns-early]]
- [[queued-messages-stranded-at-run-end]]
- [[listeners-see-stale-agent-state]]
- [[compaction-cancellation-races]]

## Related
[[turn-loop]] · [[auto-retry-backoff]] · [[overflow-recovery]] · [[auto-compaction]] · [[follow-up-queue]] · [[extension-event-hooks]] · [[agent-event-stream]] · [[cache-warming]] · [[out-of-band-message-deferral]]
