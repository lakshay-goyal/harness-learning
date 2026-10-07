---
type: concept
stage: architecture
tier: must-have
aliases: ["pi server", "pi client", "coordinator", "session-worker", "SessionWorkerManager", "Radius relay", "RadiusRelayHost", "pi-protocol", "pi-server", "pi-client", "Chord services", "CBOR framing", "PI_EXPERIMENTAL", "session-worker-isolation", "stable-coordinator-endpoint", "remote-presentation-split", "session-relay-gateway", "length-prefixed-cbor-framing", "attachment-fenced-routing", "strict-json-boundary", "control-before-bulk-scheduling", "lazy-reconnecting-connection", opencode serve, opencode attach, "http://opencode.internal", Embedded OpenCode, codex app-server, app-server-protocol, app-server-daemon, codex-app-server-client, in-process app server, ThreadManager, "--listen stdio://", serverRequest/resolved, cli-subcommand-surface]
harnesses: [pi, opencode, codex]
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
- **Process model (codex)**: one server process hosting all threads (`ThreadManager`), usable in-process via a facade, as a stdio child, or as a long-lived daemon serialized across CLI invocations and the updater ✔ codex.
- **Wire (codex)**: JSON request/notification/response/error without the `jsonrpc` field (inherited from MCP) over stdio / unix / ws / relay ✔ codex.
- **Approvals as server→client requests** resolved by whichever client answers (`serverRequest/resolved`) ✔ codex.
- **Versioning (codex)**: v1/v2 split + `experimentalApi` capability opt-in + notification opt-out at `initialize` + generated TS/JSON schemas.
- **Backpressure (codex)**: bounded commands, unbounded caller-facing event queue so responses never wait on unread events.
- **One HTTP server, every UI a client**: TUI tunnels `fetch` over worker RPC with no TCP port; web, ACP, `run`, CI and SDK use the same API (opencode).
- **Many projects per server**: instance chosen per request by directory ([[location-scoped-runtime]], opencode).
- **In-memory embedding of the same API**: router executed without a listener (opencode v2 Embedded OpenCode).

## Implementations
- [[pi--client-server-session-split|pi]] — experimental (`PI_EXPERIMENTAL=1`) coordinator/server/session-worker over Unix sockets + CBOR pi-protocol carrying Chord service calls; Radius WebSocket relay for remote clients; workers built on pi-durable.
- [[codex--client-server-session-split|codex]] — production "app-server": JSON-RPC-ish (no `jsonrpc` field) over stdio/unix/ws/remote-control, 173 client→server methods + 85 notifications, 9 server→client approval requests, capability negotiation; hosted in-process (TUI, exec), as stdio child (IDE, Python SDK) or as a managed daemon; one multi-call `codex` binary.
- [[opencode--client-server-session-split|opencode]] — `opencode serve` HTTP API + SSE; TUI in-process over worker RPC (`http://opencode.internal`); ACP and SDK as clients; v2 `packages/server` + embedded router.

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
- [[interrupt-rpc-hangs-on-finished-turn]]
- [[approval-wait-without-responder]]

## Related
[[durable-execution]] · [[replicated-state]] · [[runtime-plugin-loading]] · [[headless-rpc-mode]] · [[remote-execution-env]] · [[remote-host-trust]] · [[abort-propagation]] · [[task-owned-subagent]] · [[spec-driven-agentic-development]] · [[no-strict-jsonrpc]] · [[client-supplied-dynamic-tools]] · [[feature-flag-stages]] · [[location-scoped-runtime]] · [[sdk-embedding]]

## Tradeoffs
- [[client-server-vs-single-process]]
