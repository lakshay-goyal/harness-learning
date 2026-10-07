---
type: failure
concepts: [replicated-state, durable-execution, client-server-session-split]
harnesses: [pi, codex]
---
**Symptom** — Chord provider subscriptions invoked listeners inline while publishing (reentrant publication could reorder deliveries) and buffered updates for not-yet-activated or slow subscribers without bound.

**Root cause** — Latest-value state was treated like an event log: every delta queued per subscriber, no cap, no FIFO drain guard.

**Fix · [[pi]]** — `35180b9df` 2026-09-28 (durable document states + buffered watches): queue-for-everyone-then-drain FIFO with a `draining` guard; ≤100 pending updates, the 101st replaces the queue with one `{type:"reset", snapshot}` of `[["r", value]]` (`packages/chord/src/services/provider.ts:445-465,453-458`); public listeners keep only the newest (`state.ts:41-50`); durable watches `MAX_PENDING_WATCH_FRAMES = 100` (`packages/durable/src/session/observation.ts:15`).

**Fix · [[codex]]** — *Inverse variant* (bounded queue blocks responses): the in-process app-server client had a bounded consumer event queue; when the caller did not drain notifications the worker stalled, so request responses queued behind them never arrived. `6fc6b9d6d2` 2026-08-13 "Prevent unread events from blocking in-process requests" — caller-facing event queue made unbounded, command queues stay bounded: "the local consumer event queue is unbounded so unread notifications cannot prevent request responses from being delivered" (`codex-rs/app-server-client/src/lib.rs:14-17`). Transport side keeps an outgoing channel of 128 (`codex-rs/app-server-transport/src/transport/mod.rs:24`); per-session event channel is unbounded (`codex-rs/core/src/session/mod.rs:603-604`). Same class as [[bulk-output-starves-control-messages]].

**Lesson** — For latest-value state, bound queues by collapsing to a snapshot rather than buffering or dropping deltas; drain publications FIFO even under reentrancy. Never put control responses behind best-effort event backpressure: either collapse events (pi) or decouple the response path from the event queue (codex).

Related: [[replicated-state]] · [[durable-execution]] · [[client-server-session-split]] · [[pi--replicated-state|pi]] · [[codex--client-server-session-split|codex]]
