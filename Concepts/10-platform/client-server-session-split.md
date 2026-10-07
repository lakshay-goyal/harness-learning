---
type: concept
stage: architecture
tier: candidate
aliases: ["pi server", "pi client", "coordinator", "session-worker", "SessionWorkerManager", "Radius relay", "RadiusRelayHost", "pi-protocol", "pi-server", "pi-client", "Chord services", "CBOR framing", "PI_EXPERIMENTAL", "session-worker-isolation", "stable-coordinator-endpoint", "remote-presentation-split", "session-relay-gateway", "length-prefixed-cbor-framing", "attachment-fenced-routing", "strict-json-boundary", "control-before-bulk-scheduling", "lazy-reconnecting-connection"]
harnesses: [pi]
---
Split the agent from its UIs: each session runs in a durable worker process that owns storage and the agent loop; presentations (TUI, remote, web) attach through a routed protocol via a stable endpoint, optionally through a hosted relay for remote access.

## Why
- Sessions survive UI crashes, client disconnects and server upgrades; background work continues with no client attached.
- Multiple and remote presentations of one session (phone, web, second terminal).
- Hot-replacing server code without dropping sessions needs a stable address in front of a replaceable server.
- Wire strictness bites: `undefined` fields can't cross a strict-JSON boundary ([[strict-json-undefined-breaks-replication]]); slow peers and long socket paths, lifecycle races, relay close codes all fail ([[unix-socket-endpoint-pitfalls]], [[server-worker-lifecycle-races]], [[relay-reconnect-blocked-by-close-code]]).

## Design space
- **Process model**: single process (stable pi) vs client/server (one server) vs **coordinator → replaceable server → per-session worker** (pi experimental).
- **Wire**: JSONL (pi RPC) vs length-prefixed CBOR envelopes with strict JSON payloads (pi-protocol v8) vs WebSocket JSON.
- **Payload semantics**: domain verbs (pi-protocol v2, replaced) vs generic routed service calls + replicated state subscriptions (pi Chord, see [[replicated-state]]).
- **Versioning**: negotiation vs exact-equality version check (pi; 9 breaking versions in a month).
- **Auth layer**: in-protocol token (pi pre-`305c014dc`, rejected) vs transport-level (0600 Unix socket, relay Bearer) ([[remote-host-trust]]).
- **Routing safety**: session id only vs (serverId, sessionId, attachmentId) fencing.
- **Remote access**: inbound port vs outbound dial to a hosted relay that multiplexes clients (pi Radius).
- **Reconnect**: transparent replay vs never replay; app re-attaches (pi client).
- **Scheduling**: FIFO vs control-before-bulk priority queues (pi-env daemon) — see [[remote-execution-env]].

## Implementations
- [[pi--client-server-session-split|pi]] — experimental (`PI_EXPERIMENTAL=1`) coordinator/server/session-worker over Unix sockets + CBOR pi-protocol carrying Chord service calls; Radius WebSocket relay for remote clients; workers built on pi-durable.

## Failures
- [[strict-json-undefined-breaks-replication]]
- [[unix-socket-endpoint-pitfalls]]
- [[server-worker-lifecycle-races]]
- [[relay-reconnect-blocked-by-close-code]]
- [[rewrite-drops-test-coverage]]
- [[bulk-output-starves-control-messages]]
- [[stale-handle-reaches-new-daemon]]
- [[auth-at-message-layer]]
- [[internal-errors-leak-over-wire]]
- [[update-before-snapshot-on-subscribe]]
- [[unbounded-subscriber-buffering]]

## Related
[[durable-execution]] · [[replicated-state]] · [[runtime-plugin-loading]] · [[headless-rpc-mode]] · [[remote-execution-env]] · [[remote-host-trust]] · [[abort-propagation]] · [[task-owned-subagent]] · [[spec-driven-agentic-development]]
