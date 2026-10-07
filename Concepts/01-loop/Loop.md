---
type: group
group: 01-loop
---
Scope: the agent's control loop — turns, run settlement, mid-run input queues, loop hooks, cancellation, retry, and stream/stop-reason integrity guards that decide whether a response may drive the next step.

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
- [[step-budget-limit]] — A per-agent cap on model steps per run; the last step asks for a final answer and forbids tools by prompt or `toolChoice:none`.
- [[repeated-tool-call-detection]] — Detect the same tool call with identical input N times in a row and interrupt by asking the user or stopping.

Absence: [[no-turn-cap]].
Failures: [[Loop Failures]].
