---
type: implementation
harness: pi
concept: replicated-state
commit: b30a6dd77
files: [packages/chord/PLANNING.md:362, packages/chord/src/services/state.ts:202, packages/chord/src/services/state.ts:344, packages/chord/src/services/provider.ts:453, packages/chord/src/delta/README.md:157, packages/chord/src/delta/tracker.ts:112, packages/durable/src/session/observation.ts:15, packages/server/src/server.ts:333]
---
[[replicated-state]] in [[pi]].

## Mechanism
- **Scope**: `@earendil-works/chord` (standalone; no Pi deps; only runtime dep esbuild 0.28.2) — "Replicated state is authoritative one-writer latest-value replication" (`packages/chord/PLANNING.md:362`); non-goals "CRDT merging or multiple writers; offline mutation replay; … event history; durability" (`:389-395`). State identity = provider binding + serviceId + keyed address + member name, no state ids (`:381-387`).
- **Producer**: `MutableReplicatedState.change(context, draft => …)` — one atomic revision, synchronous callback only (async throws "must be synchronous"), reentrancy rejected; `replace(context, value)`; no-op batches not published (`packages/chord/src/services/state.ts:202-240`; `packages/chord/src/types.ts:59-71`).
- **External source**: `replicatedState(source)` attaches a `ReplicatedStateSource` (`attach() → {snapshot:{value,cursor}, activate(listener), dispose}`); Chord checks cursor contiguity, fails on gap, and "only publishes these references; it never applies or re-diffs them" (`packages/chord/src/types.ts:73-119`; `state.ts:247-341`). pi-durable plugs in here: `CommittedStateSource.attach()` (`packages/durable/src/session/observation.ts:24-81`), `task-graph.ts:88`, `view.ts:107` → durable commit line is the single writer ([[durable-execution]]). Pipeline: "hold Session mutation line → … prepare → checkpoint predicate → atomic storage commit → adopt + enqueue → release line → deliver committed ops"; "No visible-undurable path exists" (`packages/durable/docs/pico-v5-chord-usage.md:353-366`).
- **Replica** (`state.ts:344-424`): `hydrate()` requires a base batch, validates revision; `update()` requires `sequence === prev+1` else `clear()` + throw "update sequence has a gap"; gap = clear + report via binding `onError`, **no automatic resubscribe** (open decision `PLANNING.md:839`). Consumer `value` is `undefined` until hydrated / while disconnected; `subscribe(listener(value, ctx, {kind: hydrate|update, sequence}))`.
- **Back-pressure**: provider ≤100 pending updates per subscription; #101 replaces the queue with one `{type:"reset", snapshot}` whose state members are `[["r", value]]` (`packages/chord/src/services/provider.ts:453-458`; reset validated root-only `wire.ts:154-165`, `consumer.ts:683`). Public subscribers ≤100 pending complete values, overflow keeps newest — "public delivery sequences may skip. This is a frame-count policy, not a byte limit" (`state.ts:41-50`). Async listeners serialized per subscription (`state.ts:52-74`). Publications FIFO even under reentrant publish (`state.ts:149-181`; `provider.ts:445-465,482-501`).
- **Subscribe race**: provider registers subscriber before snapshot, records per-state snapshot sequences, drops buffered updates covered by the snapshot (`provider.ts:239-276,503-543`); pi-server buffers updates until the subscribe response carrying the snapshot is sent, then installs per-subscription encoder (`packages/server/src/server.ts:333-382`); client queues pre-snapshot wire updates and releases on `start()` (`packages/client/src/client.ts:172-236,313-331`).
- **Delta engine** (`@earendil-works/chord/delta`): ops `["r",v]` root replace, `["s",path,v]`, `["d",path]`, `["a",path,text]` append, `["t",path,n]` front-truncate UTF-16 units, `["p",path,i,remove,items]` splice, `["m",path,perm]` permutation (`src/delta/README.md:157-165`); "Batches are exact but not canonical". Tracker: `track` → `beginChange` → mutate overlay-Proxy draft → `prepare()` → `adopt()`; adopting one change makes competitors stale (`packages/chord/src/delta/README.md:124-136`; `tracker.ts:121-127`). "Immutability is an ownership contract. Nothing is frozen or defensively copied … an illegal mutation … silently corrupts state" (`packages/chord/src/delta/README.md:6-7`). >4096 ops ⇒ whole-root `r` (`tracker.ts:112,1707`). Reserved path segments `__proto__/constructor/prototype` never emitted; appliers throw `UnsafePathError` (`packages/chord/src/delta/README.md:170-176`; `src/delta/index.ts:133`). Wire codec interns paths "on SECOND use"; `r` resets the dictionary (`packages/chord/src/delta/index.ts:503-560`); per-(instance,member) encoders reset on snapshot/reset/replaced/unavailable (`src/services/state-codec.ts:27-136`).
- **Ownership contract instead of freezing** (`packages/chord/src/delta/README.md:28-41,183-191`): roots passed to `track()`/`replace()` move ownership to the tracker; drafts writable only while the change is open (every later use throws); `tracker.value`, prepared ops/payloads, `applyImmutable()` results and replicated values are immutable **by contract only** — "No freezing. Mutating any immutable value corrupts state silently"; loopback (in-process) consumers may share the provider's containers, so a consumer mutation corrupts authority and makes remote replicas diverge; batches passed to `apply()` are consumed (payload containers adopted). Object identity is not replicated (each path an independent placement). Trade: zero-copy speed vs. silent corruption on misuse.
- **Untrusted-side validation**: `JsonRevisionValidator` validates each incoming replica revision (strict JSON primitives, rejects cycles "Replicated state cannot contain cycles") but skips containers already validated in earlier revisions via a `WeakSet` — structural sharing makes per-revision validation O(changed) (`delta/revision-validator.ts:7-20`). Adapter-boundary check `isJsonValue()` (`src/json.ts:74`); `JsonRepresentation<T>` derives the wire-safe type of app data (`src/types.ts:26`); Chord mandates strict JSON but no framing/transport/envelope (`packages/chord/README.md:47-55`).
- **Go-like context** (`@earendil-works/chord/context`): `BACKGROUND_CONTEXT`/`TODO_CONTEXT`, typed `createContextKey`/`withContextValue` (carry permissions/telemetry without Chord depending on them), `withAbortSignal`, `withCancel`, `withoutAbortSignal` ("Intended for mandatory cleanup only"), `awaitWithContext` — cancellation rejects only the waiter, not the underlying promise (`src/context/index.ts:55-100`; `packages/chord/README.md:57-59`). Every `change(context, cb)` takes one → [[abort-propagation]].
- **Durable watches** mirror the policy: `MAX_PENDING_WATCH_FRAMES = 100` then pending suffix replaced by `[["r", newestValue]]`, retirement delivers `[["r", null]]`, "Do not use a watch as an audit log" (`packages/durable/src/session/observation.ts:15`; `pico-v5-chord-usage.md:313-320,377-380`); agent-event stream sends a fresh snapshot when a consumer is >100 batches behind (`packages/durable/src/harness/events.ts:137-156`). Late joiners start from the current view, nothing replayed (`packages/durable/README.md:302`).
- **Consumers**: experimental `Transcript` service = `{state: ReplicatedState<ConversationView>}` served from `conversation.viewState()` — "the worker keeps no reducer of its own" (`packages/coding-agent/src/experimental/services/transcript.ts:5-9`); `SessionDirectory` replicated at server scope.

## Constants
| name | value | path:line |
|---|---|---|
| provider pending updates before reset | 100 | packages/chord/src/services/provider.ts:453-458 |
| public pending deliveries | 100 (keep newest) | packages/chord/src/services/state.ts:41-50 |
| `MAX_DELTA_OPERATIONS` → root replace | 4096 | packages/chord/src/delta/tracker.ts:112 |
| `MAX_SIMPLE_OBJECT_NODES` | 128 | packages/chord/src/delta/tracker.ts:113 |
| `DEFAULT_OVERLAP_SCAN` / `MAX_IDENTITY_CANDIDATES` / `MAX_SEMANTIC_CELLS` | 65 536 / 200 000 / 65 536 | packages/chord/src/delta/diff.ts:4-5,86-87 |
| `MAX_PENDING_WATCH_FRAMES` | 100 | packages/durable/src/session/observation.ts:15 |
| durable doc checkpoint (usage guide) | `deltasSinceBase >= 99` | packages/durable/docs/pico-v5-chord-usage.md:66-67 |

## Evolution
- 2026-08-26 `a17ead329` remove `events` member kind — services are methods + state only.
- 2026-08-28 `28b49a6b3` Chord runtime foundation (moved out of `packages/agent/src/plugins/services`).
- 2026-08-31 `f7079d562`/`9af45be82` JSON delta tracking; `86bac52f9` delta-backed replicated state, per-client path codecs, server buffers until snapshot (protocol v8).
- 2026-09-01 `1a7bc80e7` service wire semantics into Chord; OpenAI `errorMessage = undefined` fix.
- 2026-09-17 `2c995acf4` delta tracker → operation log; 2026-09-18 `c4289b20e` held-reference path correctness, `58541ee72` weak proxy caches (ID-graph tracker archived: heap 430 vs 139.5 MiB, `packages/durable/docs/chord-delta-findings.md:182-192`).
- 2026-09-21 `10d1ad621` transactional `change(ctx, cb)` replaces proxy + `publish()`.
- 2026-09-23/24 `9a139c62b`, `4bc1a2fe5`, `d5cba1d97` immutable overlay tracker canonical, trusted ownership; `6966636db` `then` probe on settled draft.
- 2026-09-25 `9e70c3d50` optimized batch application; 2026-09-28 `35180b9df` document states, buffered watches, 100-update overflow → reset.

## Evidence commits
`a17ead329`, `28b49a6b3`, `86bac52f9`, `1a7bc80e7`, `2c995acf4`, `c4289b20e`, `58541ee72`, `10d1ad621`, `9a139c62b`, `6966636db`, `35180b9df`

## Quirks
- Frame-count (not byte) caps: a 16 MiB snapshot reset can still exceed the Unix transport's 64 MiB `maxPendingBytes` and disconnect the client (inferred, `packages/server/src/transports/unix/listener.ts:217-219`).
- No Chord-level version field on the wire despite `PLANNING.md:473-475`; skew handled only by pi-protocol exact version match.
- Sequence gap is effectively terminal for a subscription until the app rebinds.

## Failures
- [[strict-json-undefined-breaks-replication]] · [[held-draft-reference-misaddresses]] · [[settled-draft-then-probe-throws]] · [[delta-tracking-memory-and-size-blowup]] · [[unbounded-subscriber-buffering]] · [[update-before-snapshot-on-subscribe]]
