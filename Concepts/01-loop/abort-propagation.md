---
type: concept
stage: failure-handling
tier: candidate
aliases: [AbortController, "agent.abort()", "session.abort()", Esc, "ctx.abort()", Chord Context, withCancel, invocation-context, abortRequested, "Conversation.abort()", SessionPrompt.cancel, sessions.interrupt, Tool execution aborted, metadata.interrupted, cancelBackgroundJobs]
harnesses: [pi, opencode]
---
One cancellation signal per run threaded through the provider stream, tool executions, hooks, retry waits and side LLM work (compaction, summaries), with cancellation detected by signal state.

## Why
- Unthreaded waits ignore the user: provider streams that ignore the signal, retry sleeps, human-confirm hooks, prepared-but-unstarted tools ([[provider-stream-ignores-abort]], [[tool-preflight-ignores-abort]], [[retry-backoff-hygiene]]).
- Side phases (compaction, branch summary) are second "runs" that need the same plumbing and mutual exclusion ([[compaction-cancellation-races]]).
- Detaching a session mid-turn leaves tool calls without results unless the turn is aborted and persisted first ([[session-switch-leaves-dangling-tool-calls]]).
- Ownership trees need deterministic cascades or children outlive/miss cancellation ([[ownership-cancellation-races]]).

## Design space
- **Scope**: one controller per run (pi) · per tool · Go-style context passed to every op (pi Chord `Context`, `withCancel`).
- **Classification**: by `signal.aborted` (pi after `de2de549b`) · by error text/`AbortError` (pi before; racy).
- **Partial output**: persist aborted partial, skip on replay (pi) · discard.
- **Tools**: hard kill (process tree) · cooperative signal · in parallel batches stop preparing + short-circuit prepared thunks (pi).
- **Unprepared calls**: synthesize "No result provided" at replay (pi) · write abort results immediately.
- **Queued input on abort**: return to editor (pi) · keep queued · drop.
- **Uncancellable sections**: non-idempotent credential rotation bounded by timeout instead (pi OAuth refresh).
- **Durable**: commit abort intent → signal/join → abort owned work bottom-up → abort handler commits terminal; background tasks as boundaries (pi-durable).
- **Mechanism**: fiber interruption bridged to one `AbortController` per stream (opencode legacy) · pure Effect interruption with no signal plumbing; tools must not turn interruption into model-visible failure (opencode v2).
- **Interrupted partial output**: replay interrupted shell output as a *successful* tool result (opencode `c29392d085`) · error result.
- **Ownership cascade**: cancel background jobs transitively by parent session id before the runner (opencode).
- **Side calls**: title/summary/prune forked outside the run scope, so abort does not cancel them (opencode).

## Implementations
- [[pi--abort-propagation|pi]] — Esc → `session.abort()` (retry, compaction, branch summary, agent) → one `AbortController` → stream + tools; signal-based classification; durable ownership cascade.
- [[opencode--abort-propagation|opencode]] — legacy `Fiber.interrupt` → `AbortController` → AI SDK + tools, cleanup marks tools `interrupted`; v2 Effect interruption + uninterruptible settlement.

## Failures
- [[tool-preflight-ignores-abort]]
- [[provider-stream-ignores-abort]]
- [[compaction-cancellation-races]]
- [[session-switch-leaves-dangling-tool-calls]]
- [[retry-backoff-hygiene]]
- [[ownership-cancellation-races]]
- [[aborted-reasoning-signature-invalid]] (02-model-interface)
- [[interrupted-message-never-finalized]]
- [[orphaned-tool-calls-and-results]] (02-model-interface)

## Tradeoffs
- [[bash-timeout-default-vs-none]]
- [[turn-cap-vs-none]]

## Related
[[turn-loop]] · [[partial-message-persistence]] · [[transcript-replay-repair]] · [[process-tree-kill]] · [[steering-queue]] · [[run-settlement]] · [[task-owned-subagent]] · [[errors-as-stream-events]] · [[subscription-oauth-auth]]
