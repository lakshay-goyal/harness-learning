---
type: failure
concepts: [replicated-state, durable-execution]
harnesses: [pi]
---
**Symptom** — Chord provider subscriptions invoked listeners inline while publishing (reentrant publication could reorder deliveries) and buffered updates for not-yet-activated or slow subscribers without bound.

**Root cause** — Latest-value state was treated like an event log: every delta queued per subscriber, no cap, no FIFO drain guard.

**Fix · [[pi]]** — `35180b9df` 2026-09-28 (durable document states + buffered watches): queue-for-everyone-then-drain FIFO with a `draining` guard; ≤100 pending updates, the 101st replaces the queue with one `{type:"reset", snapshot}` of `[["r", value]]` (`packages/chord/src/services/provider.ts:445-465,453-458`); public listeners keep only the newest (`state.ts:41-50`); durable watches `MAX_PENDING_WATCH_FRAMES = 100` (`packages/durable/src/session/observation.ts:15`).

**Lesson** — For latest-value state, bound queues by collapsing to a snapshot rather than buffering or dropping deltas; drain publications FIFO even under reentrancy.

Related: [[replicated-state]] · [[durable-execution]] · [[pi--replicated-state|pi]]
