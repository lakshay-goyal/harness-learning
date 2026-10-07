---
type: group
group: 01-loop
---
Scope: the agent's control loop — turns, run settlement, mid-run input queues, loop hooks, cancellation, retry, stream/stop-reason integrity guards that decide whether a response may drive the next step, and autonomous-continuation / budget drivers.

## Concepts
- [[turn-loop]] — Inner loop: request → stream assistant → run tool calls → append results → repeat while tool calls or injected messages exist.
- [[run-settlement]] — Explicit end-of-automatic-work boundary distinct from end of one model loop; a post-run driver decides retry/compaction/continue/settle.
- [[steering-queue]] — Mid-run user messages injected at the next turn boundary without cancelling in-flight tools.
- [[follow-up-queue]] — Messages delivered only when the agent would otherwise stop.
- [[turn-lifecycle-hooks]] — Low-level callbacks to rewrite the next request or end/force a turn.
- [[abort-propagation]] — One cancellation signal per run threaded through provider stream, tools, hooks and side summarization.
- [[auto-retry-backoff]] — Harness-level retry of transient provider errors with capped exponential backoff; retryable decided by classifier.
- [[terminal-event-required]] — Streams that end without an explicit terminal event / stop reason are treated as (retryable) errors.
- [[truncated-tool-call-guard]] — Never execute tool calls from a length-truncated or unfinalized assistant message.
- [[single-active-task-slot]] — At most one task (turn, compaction, review, user shell) per session; starting one aborts the current, all share one lifecycle.
- [[mid-turn-settings-switch]] — Model/effort/permission changes during a turn captured as an immutable per-step snapshot for the next sampling request.
- [[elicitation-pause]] — Ref-counted "user is deciding" flag that pauses time-based tool limits and result delivery.
- [[session-token-budget]] — Weighted token ledger shared by a root agent tree; threshold reminders, budget-exceeded error on exhaustion.
- [[persistent-goal-continuation]] — Persisted objective + token budget; harness starts continuation turns on idle until complete/blocked/budget/breaker.
- [[persistent-agent-mode]] — Always-on mode where the model is re-sampled without new input and must pick bounded, authorized follow-ups.

Absence: [[no-turn-cap]].
Failures: [[Loop Failures]].
