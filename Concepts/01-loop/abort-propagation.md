---
type: concept
stage: failure-handling
tier: must-have
aliases: [AbortController, "agent.abort()", "session.abort()", Esc, "ctx.abort()", Chord Context, withCancel, invocation-context, abortRequested, "Conversation.abort()", SessionPrompt.cancel, sessions.interrupt, Tool execution aborted, metadata.interrupted, cancelBackgroundJobs, "Op::Interrupt", "Op::InterruptIfNoPendingInput", interrupt_task, abort_all_tasks, handle_task_abort, TurnAbortReason, CancellationToken, child_token, or_cancel, OrCancelExt, AbortOnDropHandle, "EventMsg::TurnAborted", "CodexErr::TurnAborted", finishes_on_cancellation, CleanBackgroundTerminals, "/stop"]
harnesses: [pi, opencode, codex]
---
One cancellation signal per run threaded through the provider stream, tool executions, hooks, retry waits and side LLM work (compaction, summaries), with cancellation detected by signal state.

## Why
- Unthreaded waits ignore the user: provider streams that ignore the signal, retry sleeps, human-confirm hooks, prepared-but-unstarted tools ([[provider-stream-ignores-abort]], [[tool-preflight-ignores-abort]], [[retry-backoff-hygiene]]).
- Side phases (compaction, branch summary) are second "runs" that need the same plumbing and mutual exclusion ([[compaction-cancellation-races]]).
- Detaching a session mid-turn leaves tool calls without results unless the turn is aborted and persisted first ([[session-switch-leaves-dangling-tool-calls]]).
- Ownership trees need deterministic cascades or children outlive/miss cancellation ([[ownership-cancellation-races]]).
- An abort the model cannot see makes it repeat the aborted work, incl. side effects ([[interrupted-turn-invisible-to-model]]); tearing down approval waiters before the task observes cancellation looks like a user rejection ([[approval-wait-surfaces-as-rejection-on-interrupt]]).
- "Stop the agent" ≠ "kill its long-lived processes" ([[interrupt-kills-background-processes]]); an interrupt RPC must answer even when nothing is running ([[interrupt-rpc-hangs-on-finished-turn]]).

## Design space
- **Scope**: one controller per run (pi) · per tool · Go-style context passed to every op (pi Chord `Context`, `withCancel`) · token tree task → sampling request → tool call, every await wrapped `.or_cancel(&token)` (✔ codex `CancellationToken::child_token`).
- **Grace vs hard kill**: cooperative only (pi) · cancel token → 100 ms grace for the task's `done` → hard `handle.abort()` (✔ codex).
- **Classification**: by `signal.aborted` (pi after `de2de549b`) · by error text/`AbortError` (pi before; racy).
- **Partial output**: persist aborted partial, skip on replay (pi) · discard.
- **Tools**: hard kill (process tree) · cooperative signal · in parallel batches stop preparing + short-circuit prepared thunks (pi) · abort spawned task unless the runtime `finishes_on_cancellation()` (✔ codex).
- **Unprepared calls**: synthesize "No result provided" at replay (pi) · write abort results immediately (✔ codex "aborted by user after {secs}s").
- **Background processes on interrupt**: kill process tree (✔ pi) · keep running, separate explicit kill command, tell the model (✔ codex `/stop` → `Op::CleanBackgroundTerminals`).
- **Model-visible marker**: none, aborted partial skipped on replay (pi) · persisted `<turn_aborted>` fragment, role configurable (✔ codex).
- **Teardown order**: cancel consumers first, then drop pending approvals; flush log + await extension callbacks before the terminal event (✔ codex).
- **Conditional interrupt**: only if the named turn is active and nothing is queued (✔ codex `Op::InterruptIfNoPendingInput`).
- **Queued input on abort**: return to editor (pi) · keep queued (✔ codex durable queue: held, not dispatched on interrupted idle) · drop (✔ codex in-turn pending steers cleared after cancellation) · queued agent mail may immediately start a new turn (codex).
- **Uncancellable sections**: non-idempotent credential rotation bounded by timeout instead (pi OAuth refresh).
- **Durable**: commit abort intent → signal/join → abort owned work bottom-up → abort handler commits terminal; background tasks as boundaries (pi-durable).
- **Mechanism**: fiber interruption bridged to one `AbortController` per stream (opencode legacy) · pure Effect interruption with no signal plumbing; tools must not turn interruption into model-visible failure (opencode v2).
- **Interrupted partial output**: replay interrupted shell output as a *successful* tool result (opencode `c29392d085`) · error result.
- **Ownership cascade**: cancel background jobs transitively by parent session id before the runner (opencode).
- **Side calls**: title/summary/prune forked outside the run scope, so abort does not cancel them (opencode).

## Implementations
- [[pi--abort-propagation|pi]] — Esc → `session.abort()` (retry, compaction, branch summary, agent) → one `AbortController` → stream + tools; signal-based classification; durable ownership cascade.
- [[codex--abort-propagation|codex]] — `Op::Interrupt` → `abort_all_tasks` → cancellation-token tree + `.or_cancel` everywhere; 100 ms grace then hard abort; synthetic aborted tool outputs; `<turn_aborted>` marker flushed before `TurnAborted`; background shells survive.
- [[opencode--abort-propagation|opencode]] — legacy `Fiber.interrupt` → `AbortController` → AI SDK + tools, cleanup marks tools `interrupted`; v2 Effect interruption + uninterruptible settlement.

## Failures
- [[tool-preflight-ignores-abort]]
- [[provider-stream-ignores-abort]]
- [[compaction-cancellation-races]]
- [[session-switch-leaves-dangling-tool-calls]]
- [[retry-backoff-hygiene]]
- [[ownership-cancellation-races]]
- [[aborted-reasoning-signature-invalid]] (02-model-interface)
- [[interrupted-turn-invisible-to-model]]
- [[interrupt-kills-background-processes]]
- [[approval-wait-surfaces-as-rejection-on-interrupt]]
- [[interrupt-rpc-hangs-on-finished-turn]]
- [[prewarm-blocks-turn-start]]
- [[listeners-see-stale-agent-state]]
- [[side-phase-input-lost]]
- [[turn-items-lost-on-abort]] (08-state)
- [[search-tool-stalls-on-broad-queries]] (03-tools) — grep stalled on broad searches (many matches); find could not be cancelled while discovering ignore files.
- [[interrupted-message-never-finalized]]
- [[orphaned-tool-calls-and-results]] (02-model-interface)

## Related
[[turn-loop]] · [[partial-message-persistence]] · [[transcript-replay-repair]] · [[process-tree-kill]] · [[steering-queue]] · [[run-settlement]] · [[task-owned-subagent]] · [[errors-as-stream-events]] · [[subscription-oauth-auth]] · [[single-active-task-slot]] · [[shell-execution]] · [[code-mode]]

## Tradeoffs
- [[bash-timeout-default-vs-none]]
- [[turn-cap-vs-none]]
