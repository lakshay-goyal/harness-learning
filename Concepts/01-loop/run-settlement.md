---
type: concept
stage: loop
tier: candidate
aliases: [agent_settled, agent_before_settle, _handlePostAgentRun, _runAgentPrompt, waitForIdle, willRetry, post-run driver, awaited listeners, settle boundary, on_task_finished, on_thread_idle, ThreadIdleCause, terminal_error, ThreadIdleInput]
harnesses: [pi, codex]
---
Explicit "no more automatic work" boundary, distinct from the end of one model loop; a post-run driver inspects the last response and decides retry / compaction / continue-for-queued-work / settle.

## Why
- One user prompt can span several low-level runs (retry, overflow compact-and-retry, queued follow-ups); callers that treat end-of-loop as done exit mid-retry or see stale idle state ([[retry-wait-race-prompt-returns-early]]).
- Queued work added by end-of-run handlers is stranded unless the driver re-checks queues after them ([[queued-messages-stranded-at-run-end]]).
- Fire-and-forget event handlers persist out of order; awaited listeners + a barrier make persistence part of settlement.
- "Idle" that ignores side operations (compaction, summaries) reports success while work continues ([[compaction-cancellation-races]]).
- Pending input or agent mail must not re-run a terminally failed turn ([[pending-input-restarts-failed-turn]]); a failed pre-step side phase must not lose the accepted prompt ([[compaction-drops-pending-prompt]]).
- Idle-triggered continuation drivers (queues, goals) need breakers or they run away ([[goal-continuation-runaway]]).

## Design space
- **Driver shape**: fire-and-forget handlers + promise/timer continuations (pi early, `setTimeout(100)`) · serialized event queue + retry promise (pi 2026-03) · synchronous post-run loop (pi HEAD, `32bcdc973`) · task wrapper re-running the turn while input pending, then `on_task_finished` → mailbox re-check → extension `on_thread_idle` hooks (✔ codex).
- **Who continues after idle**: in-loop queues (pi) · extensions on an idle hook calling `start_turn_if_idle` — durable queue, goal continuation; rejected rather than injected if a turn is active (✔ codex).
- **Terminal-error stop**: retry inside driver (pi) · record `terminal_error` and stop re-running despite pending input (✔ codex).
- **Settle signal**: explicit event (`agent_settled`, pi) · infer from end-of-loop + flags (`willRetry`) · polling idle.
- **Extension hook at settle**: before-settle boundary allowed one continuation (pi) · none.
- **Listener semantics**: awaited, ordered, part of idle (pi `9022a5b5e`) · observational only (pi raw `agentLoop()`) · persistence barrier: rollout flushed before `on_task_finished` and around `TurnAborted`/`TurnComplete` (✔ codex).
- **Idle definition**: no run · no run AND no compaction/summary/retry (pi).
- **Durable**: run baton document (`pi.live.run`) whose presence = busy; inputs settle `done`/`unanswered` atomically (pi-durable).

## Implementations
- [[pi--run-settlement|pi]] — `_runAgentPrompt` driver loop: retry → compaction → queued → `agent_before_settle` → `agent_settled`; `agent_end.willRetry`.
- [[codex--run-settlement|codex]] — `RegularTask` re-runs `run_turn` while input pending unless `terminal_error`; flush → `on_task_finished` → `maybe_start_turn_for_pending_work` → `on_thread_idle(cause)` extensions (queue, goal).

## Failures
- [[retry-wait-race-prompt-returns-early]]
- [[queued-messages-stranded-at-run-end]]
- [[listeners-see-stale-agent-state]]
- [[compaction-cancellation-races]]
- [[pending-input-restarts-failed-turn]]
- [[compaction-drops-pending-prompt]] (05-context)
- [[wait-misses-already-queued-result]] (09-subagents)
- [[unbounded-hook-continuation-loop]] (10-platform) — (documented hazard) An extension that unconditionally returns continue:true from turn_end /…
- [[pre-tool-hook-sees-stale-state]] (07-safety) — In multi-tool turns, tool_call handlers (policy/gate extensions) read ctx.sessionManager state that did not…
- [[tool-result-persisted-before-tool-call]] (08-state) — With async extension event handlers installed, toolResult entries were written to the session JSONL before…
- [[pre-prompt-compaction-replays-turn]] (05-context) — When the pre-prompt compaction check fired (on the last assistant message, including aborted ones), it called…

## Related
[[turn-loop]] · [[auto-retry-backoff]] · [[overflow-recovery]] · [[auto-compaction]] · [[follow-up-queue]] · [[extension-event-hooks]] · [[agent-event-stream]] · [[cache-warming]] · [[out-of-band-message-deferral]] · [[persistent-goal-continuation]] · [[single-active-task-slot]] · [[steering-queue]]
