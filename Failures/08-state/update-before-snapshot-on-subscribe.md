---
type: failure
concepts: [replicated-state, client-server-session-split]
harnesses: [pi, opencode]
---
**Symptom** — A remote subscriber could receive a state update frame before the subscribe response carrying the snapshot it applies to, so the replica either applied a delta to nothing or rejected it as a sequence gap.

**Root cause** — Snapshot (request/response path) and updates (push path) traveled independently across server and client hops with no ordering barrier.

**Fix · [[pi]]** — introduced with `86bac52f9` 2026-08-31 (delta-backed replicated state, protocol v8): the provider registers the subscriber before taking the snapshot and drops buffered updates covered by it (`packages/chord/src/services/provider.ts:239-276,503-543`); the server buffers `pendingUpdates` until the subscribe response is sent, then installs a per-subscription encoder and flushes (`packages/server/src/server.ts:333-382`); the client queues pre-snapshot wire updates and releases them on `start()` (`packages/client/src/client.ts:172-236,313-331`).

**Fix · [[opencode]]** — variant: events *lost* in the connect→subscribe window rather than reordered.
- `cb35493242` 2026-05-18 (#27959) "acquire PubSub subscription eagerly to close /event race": legacy `Bus.subscribe`/`subscribeAll` returned a `Stream` whose PubSub subscription was acquired lazily on first pull, so publishes between the `/event` SSE connect and the first pull were dropped; both became `Effect<Stream, never, Scope>` that subscribe at `yield*` time. HEAD `/event` handler registers its listener before emitting `server.connected` (`packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:28-33`).
- v2 durable tails subscribe a wake signal *before* the historical SQLite replay, then re-query rows after each wake and advance only by persisted aggregate sequence (`packages/core/src/event.ts:565-600`; `specs/v2/schema-changelog.md:579-582`), so snapshot (replay) and live updates share one cursor.

**Lesson** — Subscription hydration needs a snapshot→update barrier at every hop (provider, server, client).

Related: [[replicated-state]] · [[client-server-session-split]] · [[pi--replicated-state|pi]] · [[event-sourced-session-store]] · [[opencode--client-server-session-split|opencode]]
