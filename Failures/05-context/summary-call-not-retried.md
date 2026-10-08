---
type: failure
concepts: [auto-compaction, auto-retry-backoff, branch-summary]
harnesses: [pi]
---
**Symptom** — A transient stream drop (`terminated`, socket close) during the summarization call failed the whole compaction or branch summary on the first attempt, even though normal turns would have retried.

**Root cause** — Summary calls went straight to the provider without the session's retry policy.

**Fix · [[pi]]** — `8e53e0e49` 2026-07-21 (#6647) "compaction & branch summarization follow retry policy": `completeSummarization` wraps the call in `retryAssistantCall(produce, settings.retry, signal, callbacks)` — deterministic errors and aborts return immediately (`packages/coding-agent/src/core/compaction/compaction.ts:612-639`); `summarization_retry_*` events for the TUI (`packages/coding-agent/src/core/agent-session.ts:224-236`, `3720-3743`); backoff cap shared (`c37b0e03b` #8826). Durable compaction task has its own `retry` phase (`packages/durable/src/harness/compaction.ts:208-220`).

**Lesson** — Side LLM calls need the same transient-retry policy (and abort plumbing) as main turns.

Related: [[auto-compaction]] · [[auto-retry-backoff]] · [[branch-summary]] · [[compaction-failure-crashes-session]] · [[compaction-cancellation-races]]
