---
type: failure
concepts: [replicated-state, durable-execution]
harnesses: [pi, opencode]
---
**Symptom** — Chord provider subscriptions invoked listeners inline while publishing (reentrant publication could reorder deliveries) and buffered updates for not-yet-activated or slow subscribers without bound.

**Root cause** — Latest-value state was treated like an event log: every delta queued per subscriber, no cap, no FIFO drain guard.

**Fix · [[pi]]** — `35180b9df` 2026-09-28 (durable document states + buffered watches): queue-for-everyone-then-drain FIFO with a `draining` guard; ≤100 pending updates, the 101st replaces the queue with one `{type:"reset", snapshot}` of `[["r", value]]` (`packages/chord/src/services/provider.ts:445-465,453-458`); public listeners keep only the newest (`state.ts:41-50`); durable watches `MAX_PENDING_WATCH_FRAMES = 100` (`packages/durable/src/session/observation.ts:15`).

**Fix · [[opencode]]**
- v2 (design-time, changelog entry 2026-06-03 "Keyed Coalescing Durable Tail Signals", text landed with `76ee87ead8` 2026-06-03): the process-global unbounded aggregate-ID PubSub was replaced by one `PubSub.sliding<void>(1)` dirty signal per active tail; consumers re-query SQLite after each wake instead of queueing IDs (`packages/core/src/event.ts:565-583`; `specs/v2/schema-changelog.md:580-587` — "Slow consumers should not retain an unbounded number of redundant wake IDs").
- Still unbounded in legacy at HEAD: each `/event` SSE connection buffers into `Queue.unbounded` (`packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:30-31`) — a stalled client accumulates every instance event.

**Lesson** — For latest-value state, bound queues by collapsing to a snapshot rather than buffering or dropping deltas; drain publications FIFO even under reentrancy.

Related: [[replicated-state]] · [[durable-execution]] · [[pi--replicated-state|pi]] · [[event-sourced-session-store]] · [[opencode--event-sourced-session-store|opencode]]
