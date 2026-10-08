---
type: implementation
harness: pi
concept: client-server-session-split
commit: b30a6dd77
files: [packages/coding-agent/src/experimental/process.ts:6-105, packages/coding-agent/src/experimental/coordinator.ts:14-596, packages/coding-agent/src/experimental/server.ts:51-628, packages/coding-agent/src/experimental/session-worker-manager.ts:31-465, packages/coding-agent/src/experimental/session-worker.ts:293-594, packages/coding-agent/src/experimental/radius-relay.ts:7-689, packages/coding-agent/src/experimental/radius-auth.ts:16-66, packages/protocol/src/protocol.ts:5-104, packages/protocol/src/framing.ts:1-150, packages/protocol/src/cbor/options.ts:3-8, packages/server/src/server.ts:42-521, packages/server/src/session-router.ts:146-311, packages/server/src/transports/unix/listener.ts:11-405, packages/client/src/client.ts:76-474]
---
[[client-server-session-split]] in [[pi]].

## Mechanism
Status: experimental, `PI_EXPERIMENTAL === "1"` (`coding-agent/src/core/experimental.ts`; `experimental/commands.ts:86`); server/client/protocol are only `devDependencies` of coding-agent (`packages/coding-agent/package.json:78-80`; `1382777ed`, #9132). Stable `pi` = single process.

### Process topology
```
client TUI ──unix socket (0600)──► coordinator (stable address, opaque router)
                                     │ pipes bytes to current server endpoint
                                     ▼
                               server (replaceable generation) ──► Radius relay host (outbound WSS)
                                     │ control socket JSON lines
                                     ▼
                       session-worker per Session (lock + session.sqlite + pi-durable Harness)
```
- Roles via env `__PI_INTERNAL_SPAWN`, consumed so descendants don't inherit; spawned `detached`, `stdio:"ignore"`, `unref()` (`experimental/process.ts:6-33,54-70`); control frames = JSON lines ≤ `MAX_CONTROL_LINE_BYTES` 128 MiB (`process.ts:100-105`).
- **Coordinator** ("intentionally opaque message router", `coordinator.ts:42`), `COORDINATOR_PROTOCOL_VERSION = 3` (`:14`): public socket piped to current server endpoint (`:444-473`); new server registration → previous gets `server_replaced`, public connections closed (`:372-397`) = hot server replacement behind a stable address; only the server may broadcast (`:435`); sockets `chmod 0o600` (`:570-572`); stale sockets detected by test-connect (`:574-596`); exits when empty after 30 s startup grace / 250 ms empty grace (`:263-264,483-500`).
- **Server** (`experimental/server.ts`): dir `~/.pi/server` or `PI_SERVER_DIR` (`:51-58`); logical server-id lock `LOCK_STALE_MS=30_000` (`:73`); sockets `control-<id>.sock`, `server-<id>-<nonce>.sock` (`:582-586`); "never opens the storage itself" (`48dd1e2f0`); sessions = dirs with `meta.json` + `session.sqlite` (`session-catalog.ts:5,15-16`).
- **SessionWorkerManager** ("Session and process bookkeeping owned by one replaceable server process", `session-worker-manager.ts:94`): timeouts startup 15 s / shutdown 10 s / discovery 5 s / demand 5 s (`:31-34`); new server generation rediscovers live workers via `discover_workers` broadcast (`:150-175`) → workers survive server restarts; attach/release = "demand" messages (`:233-315`); worker ops have no wall-clock timeout (`:317`); messages `operation / operation_cancel / session_demand / shutdown` (`:305,340,352,382`).
- **Session worker** (`session-worker.ts`): `proper-lockfile` on session dir (stale 2 s, update 1 s, ~8 s retries; 320 × 25 ms) (`:517-522`), opens Harness, subscribes `harness.taskGraph()`, calls `harness.resume()` ("Recovered work from an interrupted turn continues now", `:593-594`). `WorkerLifecycle` retires only when demand initialized, no retirement holds, no live Harness tasks, no attachments (`:293-305`) — "Every live task, background work included, keeps the worker and its Session open." (`:591`). Grace 10 s initial / 30 s orphan demand (`:307-308`). Server↔worker auth via `PI_SESSION_WORKER_CONTROL_TOKEN` + `PI_SESSION_WORKER_SESSION_KEY_BASE64`, each control message carries `token`/`sessionKey` (`:64-67,536-539`).
- **Services** on Chord facets (`experimental/services/README.md:14-26`): server scope `SessionDirectory` (replicated), `SessionManagement`, `PresentationPlugins`; session scope `SessionPlugins`, `Models`, `AgentController` (prompt/steer/follow-up/abort/compact facade over root conversation), `Transcript` = `{state: ReplicatedState<ConversationView>}` from `conversation.viewState()` — "the worker keeps no reducer of its own" (`services/transcript.ts:5-9`); presentation scope `SlashCommands`, `PresentationUI`. Worker builds one `FacetHost` and exposes per scope via `createRemoteServiceEndpoint` (`services/worker.ts:82,128`). Client binds with `createClientServiceTransport(client, () => client.attachment)`, `rebind()` on attachment change; session source `acceptsUnavailableServices` while detached (`services/connection.ts:62-66,109-124,198-206,229,317-321`).
- Known gaps (`services/README.md:38-45`): tree navigation dropped ("Durable branches by forking"), next-run queue and `resume()` dropped, subagents not exposed, transcript paging TODO; experimental durable TUI lacks session picker, forks/tree, extensions, prompt templates, images, `/login` (`experimental/durable/README.md:66`).

### Wire: `pi-protocol` (CBOR)
- "Runtime-neutral routed envelopes, CBOR encoding, and byte-stream framing"; "experimental and has no compatibility guarantees" (`packages/protocol/README.md:3,41`).
- `PROTOCOL_VERSION = 8` (`protocol.ts:5`); exact-equality check, no negotiation (`codec.ts:139-141`; hello schema `Type.Literal(PROTOCOL_VERSION)` `protocol.ts:65-69`); mismatch → `hello_error {code:"version"}` (`server/src/server.ts:263-268`).
- Framing: 4-byte BE length + exactly one definite-length CBOR item (`framing.ts:1,27-39`); `DEFAULT_MAX_FRAME_LENGTH` 16 MiB, hard cap `0xffff_ffff`; 64 KiB accumulation blocks; oversize → decoder permanently `failed`; truncated stream at `end()` fails (`framing.ts:2-6,77-150`).
- Hand-written RFC 8949 subset: max 16 MiB bytes / 1 000 000 container entries / depth 64 (ceiling 512) (`cbor/options.ts:3-8`); rejects tags, indefinite lengths, half/single floats (float64 only), non-finite, unsafe ints, non-string or duplicate map keys, invalid UTF-8, trailing data (`decoder.ts:22-139`); map entries via `Object.defineProperty` (prototype-pollution safe, `:70-75`); encoder rejects cycles/holes/lone surrogates, silently drops `undefined` map values, insertion-order keys (`encoder.ts:113-200`).
- Protocol stricter than CBOR: TypeBox schema AND Chord `isJsonValue` (no byte strings, `undefined`, non-finite, cycles) (`codec.ts:20-32`); errors truncated to 500 chars; failed decoder stays failed (`codec.ts:34-91`).
- Messages: client → `hello {version}` (first), `request {id, target, call}`, `cancel {id, target}`; server → `hello {version, serverId}`, `hello_error`, `response {id, ok, result|error{code,message}}`, `service_update {subscriptionId, update}`, `attachment {SessionTarget|null}` (`protocol.ts:28-104`). Targets: `{serverId}` or `{serverId, sessionId, attachmentId}`; `serverId` canonical lowercase UUIDv4; all objects `additionalProperties:false` (`protocol.ts:9-47`). `call/result/update` are opaque JSON owned by Chord (`packages/protocol/README.md:15`).
- Auth: "Peer authentication and authenticated service contexts are not implemented by the experimental transport" (`packages/protocol/README.md:39`); framed `auth {token}` removed in `305c014dc` → transports must be pre-authorized (0600 sockets, relay Bearer) → [[remote-host-trust]].

### `pi-server` / `pi-client`
- Server routes opaque service envelopes, doesn't load facet contracts (`packages/server/README.md:3-13`). Abstract `ServerListener.start(accept)` → `ByteConnection` (`listener.ts:3-8`); shipped transport = Unix sockets only.
- Unix listener: mode `0o600`, parent dir `0o700`, path `<dir>/<serverId>.sock` (`transports/unix/listener.ts:11,53`; `address.ts:4-8`); atomic bind via private `bind-<sha256(path)[:8]>` → `lstat` dev/ino → `link()` → chmod → unlink (`listener.ts:52-80,294-297`); stale sockets probed 1 s and removed only if dead and same inode (`:15,299-336`); per-connection `maxPendingBytes` default 4 × frame = 64 MiB, overflow rejects send → server closes connection (`:212-228,402-405`; `packages/server/src/server.ts:463-471`); graceful close 5 s.
- Handshake: first message `hello`, 5 s timeout (`packages/server/src/server.ts:42,147-153`); duplicate hello → `invalid_request`; requests during handshake queued (`:239-259`).
- Requests: duplicate active id → error; `parseServiceCall` → session targets → `SessionRouter.executeServiceCall`, server targets → `serverServices.invokeService`; wrong server → `WrongServerError` (`packages/server/src/server.ts:306-404`). Subscribe: server buffers updates until the snapshot response is sent, then per-subscription `createServiceStateEncoder()` and flush (`:333-382`, `86bac52f9`). Cancel aborts by target (`:298-304`) → `{code:"cancelled"}`.
- Error sanitization: `ServerError`/`RemoteServiceError` codes pass; validation → `invalid_request`; anything else → `internal_error` "Internal server error", logged locally (`packages/server/src/server.ts:512-521`; `dab70b2cd`). Codes `wrong_server, session_not_found, session_ambiguous, session_not_attached, server_draining` (`server/src/errors.ts:3-9`).
- **Attachment fencing** (`session-router.ts`): one live attachment per connection; attach idempotent for same session (`:160-163`); `attachmentId = randomUUID()` (`:167-172`) published out-of-band (`:192-196`); session calls must match `sessionId` AND `attachmentId` else `SessionNotAttachedError` (`:224-232`) — fences delayed frames after re-attach; per-client ops serialized (`:146-158`); release waits for admitted ops (`:234-252`); concurrent opens share one promise (`:262-274`); handle `terminated` invalidates attachments (`:293-311`). "Neither an open JavaScript Session nor a Harness crosses the process boundary" (`server/README.md:57`).
- Client: canonical UUIDv4 `serverId` required; handshake verifies endpoint's `serverId` (`client/src/client.ts:76-79`; `packages/client/src/connection.ts:174-181`); no client-side handshake timer (unverified intent); `request()` ids `request-<n>`, abort → local reject + `cancel`; pre-snapshot updates queued then decoded after snapshot, released on `start()`; decode error fails connection; disconnect rejects pending, clears attachment, drops subscriptions; "It never reconnects or replays requests automatically… explicitly repeat only operations known to be safe" (`packages/client/src/client.ts:172-474`; `packages/client/README.md:34`). Unix discovery: scan dir for `*.sock`, ≤16 concurrent probes, 1 s timeout (`client/src/unix.ts:14-17`).
- Browser smoke entry imports `pi-client`/`pi-protocol` (`scripts/browser-smoke-entry.ts:1,5`; `56eb685b6`).

### Radius relay (remote access)
- Radius = Earendil's hosted gateway (`DEFAULT_RADIUS_GATEWAY = "https://radius.pi.dev"`, `packages/ai/src/providers/radius-config.ts:4`; override `PI_RADIUS_GATEWAY`, `core/radius.ts:3-11`); also AI gateway, MCP endpoint, bug reports, `/share` artifacts.
- Topology: local server = **host**, dials `wss://<gateway>/v1/session-relays/<serverId>/connect` subprotocol `pi-session-relay.host.v1`; clients dial same URL with `pi-session-relay.client.v1` (`radius-relay.ts:7-8,618-624`) → no inbound port (gateway closed source, pairing inferred, unverified). Client checks server selected exactly the requested subprotocol (`:662-667`); target `--connect radius://<uuid>` (`cli/experimental/command-options.ts:47-63`).
- Auth: `Authorization: Bearer <token>` on upgrade (`radius-relay.ts:626-646`); `RadiusRelayAuthResolver.resolve` per attempt: `PI_OFFLINE` disables → `--auth-token`/`--auth-token-file` (exclusive) → stored `radius` OAuth refreshed if <5 min validity; required & missing → "run `/login radius`" (`radius-auth.ts:16-66`).
- Host multiplexing: text JSON control `{version:1, type: ping|pong|connection_open|connection_close, connection_id, code?}` strictly validated (`:538-590`); binary data `u8 version=1 | u8 type=1 | 16-byte connection UUID | payload` = 18-byte header (`:10-13,592-616`); `connection_open` → `RelayServerByteConnection` handed to the same `Server.accept()` stack; reused id = protocol error; unknown id → `connection_close` (`:243-268`).
- Client: dedicated WebSocket, binary only (`RadiusClientByteTransport`, `:437-494`); can't pass plugin package paths through Radius (`client-runtime.ts:66-68`).
- Flow control: `OrderedWebSocketWriter` serializes sends; rejects when pending > 4 × 16 MiB = 64 MiB; waits in 5 ms sleeps while `bufferedAmount` > 1 MiB (`radius-relay.ts:14-15,496-536`).
- Reconnect: host 1 s → 30 s backoff; no auth → `not_authenticated`, retry every 30 s, server stays "local only" (`:16-20,135-172`; `packages/coding-agent/src/experimental/commands.ts:15-26`); client `RadiusClientReconnect` 1 s → 30 s and re-attaches last session via `SessionManagement.attach` (`:343-401`; `client-runtime.ts:156-168`).
- Close codes: undici browser-style WS only allows sending 1000 or 3000–4999 → local errors 4000 (protocol) / 4001 (transport) (`:21-25,683-689`; `35c49350f`).
- Started for every experimental foreground server, closed with server or coordinator replacement (`experimental/server.ts:609-628`). No E2E encryption above TLS — gateway sees traffic (no crypto in `radius-relay.ts`).

## Constants
| name | value | path:line |
|---|---|---|
| `PROTOCOL_VERSION` | 8 | `packages/protocol/src/protocol.ts:5` |
| `DEFAULT_MAX_FRAME_LENGTH` | 16 MiB | `packages/protocol/src/framing.ts:6` |
| CBOR byte / container / depth | 16 MiB / 1 000 000 / 64 (≤512) | `packages/protocol/src/cbor/options.ts:3-8` |
| handshake timeout | 5 000 ms | `packages/server/src/server.ts:42` |
| Unix close / probe timeout | 5 s / 1 s | `packages/server/src/transports/unix/listener.ts:12,15` |
| `maxPendingBytes` default | 4 × frame (64 MiB) | `listener.ts:212-228` |
| client discovery timeout / probes | 1 s / 16 | `packages/client/src/unix.ts:14,17` |
| `COORDINATOR_PROTOCOL_VERSION` / start timeout / retry | 3 / 10 s / 10 ms | `experimental/coordinator.ts:14-16` |
| coordinator empty grace | 30 s startup / 250 ms | `coordinator.ts:263-264` |
| server lock stale / retry / wait | 30 s / 25 ms / 30 s | `experimental/server.ts:73-75` |
| auto-server startup / idle grace | 10 s / 1 s | `experimental/server.ts:244-245` |
| worker startup/shutdown/discovery/demand | 15 / 10 / 5 / 5 s | `session-worker-manager.ts:31-34` |
| worker demand grace | 10 s initial / 30 s orphan | `session-worker.ts:307-308` |
| worker lock retries | 320 × 25 ms (~8 s) | `session-worker.ts:521` |
| `MAX_CONTROL_LINE_BYTES` | 128 MiB | `experimental/process.ts:100` |
| relay backoff | 1 s → 30 s | `radius-relay.ts:16-20` |
| relay drain threshold | 1 MiB | `radius-relay.ts:15` |
| relay header | 18 bytes | `radius-relay.ts:10-13` |
| relay close codes | 4000 / 4001 | `radius-relay.ts:21-25` |

## Evolution
- Pre-history: `packages/orchestrator` from 2026-06-18 (`7ece19b0e`, `d799c7220` ipc socket, `92e28e9c5` ipc server; 53 commits) → renamed server `8495f9d0d` 2026-07-21 (#6898).
- 2026-07-30 `56eb685b6` pi-protocol born: CBOR + framing + domain verbs (`list/create/attach/detach/prompt/steer/abort/set_model/set_thinking`, snapshot/progress events) at v2; `bd2cfabc5` reject cycles.
- 2026-07-31 `33bc0a7b8`, `7e121c67c`, `ee3156715`, `1c418f624` runtime-neutral client, Unix transport (Bun too); `2e60d3cbd`, `cb86f0844`, `91957b3a0` composable server, adapter invariants. 2026-08-01 `a3ec51d2a` remote session coordination.
- 2026-08-03 `305c014dc` removed framed bearer auth, version 2→1 (#7551); 2026-08-04 `05bf9df65` deleted legacy orchestrator (−2,012 lines); 2026-08-05 `dab70b2cd` sanitize failures (#7644).
- 2026-08-13 `e52de91d0` service-addressed session RPC (−6,277 lines), `c40e08b8b`, `747914847` discovery; 2026-08-14 `ce142b2a6` restored test coverage; `17fd50262` sessions in child processes; `6b3fb49dd` fail-fast server generation replacement; `9fcaac9b1` bounded socket paths; `80b6797c9` UUIDv4 server ids.
- 2026-08-15 `8dce5da76` coordinate replaceable servers and workers; 2026-08-16 `33dac3623` runtime moved outside CLI; `fc0a7cb36` lifecycle races.
- 2026-08-17–20 `a8f856a9a`, `e761b0d0f`, `f13dbebb4`, `fb33e0db2` remote workers / one-shot prompts / streamed events / remote session RPC; `f8a6e670d` replace raw Session RPC with routed services through fenced attachments (v1→2); `8222adee8` routed plugin services (v3).
- 2026-08-26 `a17ead329` removed `events` member kind; `353c990f4` "mini" three-process agent ("built to find out what an RPC-shaped presentation actually needs… A presentation holds a replicated LaneSnapshot and nothing else").
- 2026-08-28 `1d0d110ab` Radius relay (`radius-relay.ts` 704 lines, +1308); `35c49350f` reconnect after abnormal closure; `28b49a6b3` Chord foundation. 2026-08-30 `984846b78` client re-attach via injected `reattach`; 2026-08-31 `8e374e4dc` reconnect narrowed to `Pick<Client,…>`.
- 2026-08-30/31 `0252dff88`, `429f4e756`, `5245aba7c`, `ae2cc5116`, `86bac52f9` facet distribution, package plugins, delta-backed replicated state (v4→8); 2026-09-01 `1a7bc80e7` wire semantics into Chord, payloads opaque.
- 2026-10-01 `48dd1e2f0` client/server ported onto pi-durable; pi-server drops pi-agent-core dep; v1.0.0. `1382777ed` remote deps dev-only (#9132).
- Protocol versions (`-G 'PROTOCOL_VERSION = '`): `56eb685b6`(2) → `305c014dc`(1) → `f8a6e670d`(2) → `8222adee8`(3) → `0252dff88`(4) → `429f4e756`(5) → `5245aba7c`(6) → `ae2cc5116`(7) → `86bac52f9`(8): nine wire-breaking versions in one month.

## Evidence commits
`56eb685b6` `305c014dc` `05bf9df65` `dab70b2cd` `e52de91d0` `ce142b2a6` `17fd50262` `6b3fb49dd` `9fcaac9b1` `8dce5da76` `fc0a7cb36` `f8a6e670d` `8222adee8` `353c990f4` `1d0d110ab` `35c49350f` `984846b78` `86bac52f9` `1a7bc80e7` `48dd1e2f0` `1382777ed`

## Quirks
- No Chord-level version field although `PLANNING.md:473-475` requires one; skew relies on exact `PROTOCOL_VERSION` only.
- Server→client carries only updates, no calls (symmetric RPC planned, `PLANNING.md:399-439`); context values don't cross the wire except abort (`packages/server/src/server.ts:331` uses `TODO_CONTEXT`).
- A 16 MiB snapshot reset can exceed `maxPendingBytes` and disconnect the client (inference, `listener.ts:217-219`).
- Relay client `assertAccess() {}` is a no-op client-side (`client-runtime.ts:158-160`); gateway authorization of `serverId` unverified.
- Can a worker with a `missing_task` blocked task ever retire? (open; `session-worker.ts:591`, `spec.md:2027-2031`).
- Shutdown order matters: detach clients → dispose facet host → close Harness, else clients keep a frozen last value (`packages/durable/docs/pico-v5-chord-usage.md:172-183`).

## Failures
[[strict-json-undefined-breaks-replication]] · [[unix-socket-endpoint-pitfalls]] · [[server-worker-lifecycle-races]] · [[relay-reconnect-blocked-by-close-code]] · [[rewrite-drops-test-coverage]] · [[bulk-output-starves-control-messages]] · [[stale-handle-reaches-new-daemon]] · [[auth-at-message-layer]] · [[internal-errors-leak-over-wire]] · [[update-before-snapshot-on-subscribe]] · [[unbounded-subscriber-buffering]] · [[held-draft-reference-misaddresses]] · [[delta-tracking-memory-and-size-blowup]]
