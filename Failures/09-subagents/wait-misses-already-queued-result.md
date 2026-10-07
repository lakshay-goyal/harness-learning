---
type: failure
concepts: [subagent-result-mailbox, in-process-subagent-threads, task-owned-subagent, run-settlement]
harnesses: [codex]
---
**Symptom** — `wait_agent` subscribed only to *future* mailbox changes; if a child's result was queued before the wait started, the parent slept the full `timeout_ms` although results were ready.

**Root cause** — Classic lost wakeup: subscribe-then-wait without checking current state.

**Fix · [[codex]]**
- `639382609f` 2026-04-22 "fix: wait_agent timeout for queued mailbox mail (#18968)" — check `has_pending_mailbox_items()` first; `subscribe_activity` returns `pending_activity` (`codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs`).
- Same family: `e2551a5e36` 2026-05-28 "Reap stale multi-agent slots" (close on a dead child errored and leaked the slot); `acc20df49f` / `960e878df4` eviction vs queued mail ([[subagent-eviction-loses-mail]]).

**Lesson** — Any "wait for event" primitive must check current state after subscribing, not only future notifications; close must be idempotent on dead children.

Related: [[subagent-result-mailbox]] · [[in-process-subagent-threads]] · [[task-owned-subagent]] · [[run-settlement]] · [[codex--subagent-result-mailbox|codex]]
