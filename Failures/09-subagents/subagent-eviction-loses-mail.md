---
type: failure
concepts: [subagent-concurrency-limits, subagent-result-mailbox]
harnesses: [codex]
---
**Symptom** — Unloading an idle V2 agent for capacity could drop accepted-but-unprocessed messages; unread mail also kept idle agents loaded, and messaging an evicted agent reloaded it.

**Root cause** — The inbox lived inside the worker session; eviction teardown raced queued submissions and could be abandoned by cancellation.

**Fix · [[codex]]**
- `acc20df49f` 2026-09-22 "Prevent V2 agent eviction from racing with queued messages" — pin recipients until queued submissions are handled; teardown in a separate task so cancellation can't abandon it.
- `960e878df4` 2026-10-01 "Preserve queued agent mail across session eviction" — runtime mailbox outlives the session (`codex-rs/core/src/agent/control/mailbox.rs:1-60`).
- Related: `e2551a5e36` 2026-05-28 "Reap stale multi-agent slots" (`close_agent` on a dead thread errored and leaked the slot).

**Lesson** — If workers can be unloaded for capacity, their inbox must live outside the worker and eviction must not race delivery.

Related: [[subagent-concurrency-limits]] · [[subagent-result-mailbox]] · [[wait-misses-already-queued-result]] · [[codex--subagent-concurrency-limits|codex]]
