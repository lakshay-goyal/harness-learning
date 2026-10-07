---
type: failure
concepts: [replicated-state, client-server-session-split]
harnesses: [pi]
---
**Symptom** — A remote subscriber could receive a state update frame before the subscribe response carrying the snapshot it applies to, so the replica either applied a delta to nothing or rejected it as a sequence gap.

**Root cause** — Snapshot (request/response path) and updates (push path) traveled independently across server and client hops with no ordering barrier.

**Fix · [[pi]]** — introduced with `86bac52f9` 2026-08-31 (delta-backed replicated state, protocol v8): the provider registers the subscriber before taking the snapshot and drops buffered updates covered by it (`packages/chord/src/services/provider.ts:239-276,503-543`); the server buffers `pendingUpdates` until the subscribe response is sent, then installs a per-subscription encoder and flushes (`packages/server/src/server.ts:333-382`); the client queues pre-snapshot wire updates and releases them on `start()` (`packages/client/src/client.ts:172-236,313-331`).

**Lesson** — Subscription hydration needs a snapshot→update barrier at every hop (provider, server, client).

Related: [[replicated-state]] · [[client-server-session-split]] · [[pi--replicated-state|pi]]
