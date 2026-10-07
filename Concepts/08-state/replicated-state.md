---
type: concept
stage: state
tier: variant
aliases: [Chord ReplicatedState, MutableReplicatedState, replicatedState(), ReplicatedStateSource, json-delta-ops, "change(ctx, draft)", watch, viewState, copy-on-write-draft-transaction, committed-view-replication, snapshot-then-buffered-updates, overflow-collapses-to-snapshot]
harnesses: [pi]
---
Agent/UI state published by a single authoritative writer as sequence-numbered revisions — a snapshot on subscribe then exact delta-op batches — to local or remote replicas, with bounded buffering that collapses to a fresh snapshot on overflow; explicitly not a CRDT.

## Why
- Multiple presentations (TUI, remote client, web) need the same live view of a worker-owned session without each re-deriving it or shipping whole transcripts per token.
- Subscribe races (update before snapshot), sequence gaps and slow consumers must have defined behavior ([[update-before-snapshot-on-subscribe]], [[unbounded-subscriber-buffering]]).
- Strict-JSON wire makes `undefined` vs absent a correctness bug ([[strict-json-undefined-breaks-replication]]).

## Design space
- **Event stream reducers in every client** (pi stable `AgentSessionEvent`) vs **replicated latest-value state** (Chord) vs **CRDT multi-writer** (rejected: "not a CRDT", non-goal).
- **Producer API**: mutable proxy + explicit publish (early Chord) vs transactional `change(ctx, draft => …)` (since `10d1ad621`) vs external committed source (`ReplicatedStateSource`, pi-durable).
- **Delta encoding**: diff of values vs operation log vs immutable overlay tracker (pi settled on overlay; ID-graph tracker rejected on memory).
- **Ops**: `r/s/d/a/t/p/m` tuples, path interning on second use, >4096 ops ⇒ root replace.
- **Back-pressure**: frame-count cap 100 → collapse to `reset` snapshot (provider) / keep newest (public listeners) vs byte limits vs disconnect.
- **Gap handling**: clear + error (pi, no auto-resubscribe) vs resync.
- **Visibility**: only committed state replicated ("No visible-undurable path") in pi-durable.
- codex: absent — clients reduce an event stream (app-server `thread/started → turn/started → item/started → item/*/delta → item/completed → turn/completed`, `codex-rs/app-server-protocol/src/protocol/common.rs:1935-2030`) and re-read via `thread/read`/`thread/turns/list`; no snapshot+delta replicated state. Backpressure choice is the opposite: unbounded caller-facing event queue so responses are never blocked ([[codex--client-server-session-split|codex client-server-session-split]], [[unbounded-subscriber-buffering]]).

## Implementations
- [[pi--replicated-state|pi]] — `@earendil-works/chord` replicated state + delta engine; pi-durable `viewState()/watch()` as committed source; server buffers updates until snapshot response; 100-update overflow → reset.

## Failures
- [[strict-json-undefined-breaks-replication]]
- [[held-draft-reference-misaddresses]]
- [[settled-draft-then-probe-throws]]
- [[delta-tracking-memory-and-size-blowup]]
- [[unbounded-subscriber-buffering]]
- [[update-before-snapshot-on-subscribe]]

## Related
[[durable-execution]] · [[client-server-session-split]] · [[agent-event-stream]] · [[runtime-plugin-loading]] · [[headless-rpc-mode]]
