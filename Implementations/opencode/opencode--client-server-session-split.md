---
type: implementation
harness: opencode
concept: client-server-session-split
commit: ecc4916b5a
files: [packages/opencode/src/cli/cmd/tui.ts:199-250, packages/opencode/src/cli/network.ts:15-73, packages/opencode/src/server/routes/instance/httpapi/groups/, packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:40-75, packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:87, packages/opencode/src/cli/cmd/acp.ts:20-60, packages/sdk/js/src/index.ts:8, packages/server/src/api.ts:1-5, CONTEXT.md:139-170]
---
[[client-server-session-split]] in [[opencode]].

## Mechanism
### Legacy runtime
- **One server, every UI a client**: TUI, web/desktop app, ACP bridge, `opencode run`, the GitHub agent, remote workspaces and the JS SDK all speak the same HTTP API. Route groups: config, control-plane, event, experimental, file, global, instance, mcp, permission, project, provider, pty, question, session, sync, tui, workspace (`packages/opencode/src/server/routes/instance/httpapi/groups/`).
- **TUI without a port**: the server runs in a Worker thread; the TUI's SDK client gets `url: "http://opencode.internal"` with a `fetch` tunneled over worker RPC and an RPC event source (`packages/opencode/src/cli/cmd/tui.ts:199-250`). A real listener starts only with `--port`/`--hostname`/`--mdns`. TUI is `@opentui/solid`; the Go Bubbletea TUI was deleted `f68374ad22` (2025-11-02).
- **Event streaming**: `GET /event` SSE of the instance bus, starts with `server.connected`, `server.heartbeat` every 10 s, ends on `server.instance.disposed` (`packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:40-75`); `/global/event` carries all instances plus `sync` events for replication ([[agent-event-stream]]).
- **Human-in-the-loop over the wire**: `permission.asked` / `question.asked` events, answered by POST routes; the UI is entirely client-side ([[permission-ruleset]]).
- **Multi-project**: each request picks its instance by `?directory=` → `x-opencode-directory` → `process.cwd()` (`packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:87`) ([[location-scoped-runtime]]).
- **Auth/CORS**: optional Basic auth, `127.0.0.1` default bind, origin allowlist ([[remote-host-trust]]).
- **ACP**: `opencode acp` starts a real `Server.listen`, builds an SDK client against it, and bridges stdin/stdout NDJSON to `AgentSideConnection` (`packages/opencode/src/cli/cmd/acp.ts:20-60`) ([[headless-rpc-mode]]).
- **Remote workspaces**: requests for a remote workspace are proxied to another opencode server; its events replicated back through `/global/event` + `/sync/history` ([[remote-execution-env]]).

### v2 runtime
- `packages/server` re-implements handlers over `packages/protocol` `makeDefaultApi`, sharing auth and CORS helpers (`packages/server/src/api.ts:1-5`).
- Networked and **Embedded OpenCode** clients share one `HttpApi` contract; the embedded SDK "executes Server's assembled `HttpRouter` in memory. It opens no listener" (`CONTEXT.md:162`) ([[sdk-embedding]]).
- Durable per-session event stream with `after` cursor vs live instance stream without replay; neither auto-reconnects (`CONTEXT.md:156`, `CONTEXT.md:165-169`).

## Constants
| name | value | path:line |
|---|---|---|
| SSE heartbeat | `Stream.tick("10 seconds")` | `packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:63` |
| HTTP compression threshold | 1024 bytes | `packages/opencode/src/server/routes/instance/httpapi/middleware/compression.ts:14` |
| TUI event batch flush | 16 ms | `packages/tui/src/context/sdk.tsx:72-79` |

## Evolution
- 2025-09-01 `f993541e0b` multiple instances inside one process.
- 2025-10-20 `f3f21194ae` ACP support.
- 2025-11-02 `f68374ad22` Go TUI deleted.
- 2026-01-12 `1954c1255e` password auth.
- 2026-05-18 `cb35493242` `/event` subscription acquired eagerly; events between connect and subscribe were lost ([[update-before-snapshot-on-subscribe]]).
- 2026-06-03 `76ee87ead8` embedded v2 runtime in the same server layer graph; 2026-06-26 `a1f093a748` legacy stops emitting v2 events.

## Quirks / drift
- The legacy runtime still owns the shipping loop; the HTTP server provides both legacy services and `SessionV2` in one layer graph (`packages/opencode/src/server/routes/instance/httpapi/server.ts:298-303`).

pi contrast: pi's experimental split puts each session in its own worker process behind a coordinator, CBOR over Unix sockets, relay for remote ([[pi--client-server-session-split|pi]]).
