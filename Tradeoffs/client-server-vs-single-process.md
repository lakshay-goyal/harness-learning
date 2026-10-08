---
type: tradeoff
concepts: [client-server-session-split, headless-rpc-mode, sdk-embedding, location-scoped-runtime]
harnesses: [pi, opencode]
---
# client-server-vs-single-process

**Axis**: is the agent a single process the UI lives in, or a server that every UI (TUI, web, IDE, SDK) talks to?

| dimension | pi | opencode | evidence |
|---|---|---|---|
| Stable architecture | single process; TUI and session in one Node process | server always: the TUI runs the server in a Worker thread and talks HTTP-shaped `fetch` over worker RPC to `http://opencode.internal`; no TCP port unless asked | pi → [[client-server-session-split]] (stable = single process); opencode `packages/opencode/src/cli/cmd/tui.ts:22-56,199-256` |
| Programmatic access | `--mode rpc` JSONL over stdin/stdout (33 commands); in-process SDK | HTTP API + generated SDK (`@opencode-ai/sdk`, hey-api from OpenAPI); ACP over stdio for editors | pi `packages/coding-agent/src/modes/rpc/rpc-types.ts:22-74` → [[headless-rpc-mode]], [[sdk-embedding]]; opencode `packages/sdk/js/script/build.ts:10-16`; `packages/opencode/src/acp/agent.ts:35-80` |
| Experimental server | `PI_EXPERIMENTAL`: coordinator + replaceable server + per-session worker over a unix socket (0600) | — (already the default) | pi `packages/coding-agent/src/experimental/process.ts:6-105` |
| Multi-directory hosting | one cwd per process | one process hosts many directories; each gets a lazily built service graph, evicted after 60 min idle | `packages/core/src/location-services.ts:109` → [[location-scoped-runtime]] |
| Network exposure | n/a in stable | 127.0.0.1 by default; `--mdns` binds 0.0.0.0; Basic auth only if `OPENCODE_SERVER_PASSWORD` is set, else "server is unsecured" warning | `packages/opencode/src/cli/network.ts:15-79`; `packages/opencode/src/server/auth.ts:18-41`; `SECURITY.md:21-23` |
| Event delivery | in-process listeners | SSE with 10 s heartbeat; TUI batches events within 16 ms | `packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:63`; `packages/tui/src/context/sdk.tsx:72-79` |

**When each wins**
- **Single process (pi)**: simplest lifecycle (everything dies with the terminal), no auth surface, lowest latency; RPC mode covers embedding.
- **Server-first (opencode)**: one session viewable from TUI, desktop, web and IDE at once; remote/attach workflows; sharing and CI drive the same API. Costs: an HTTP surface to secure (unauthenticated by default when exposed), state that must be durable and ordered across clients ([[session-store-format]], [[mid-run-user-input]]), and more moving parts per session.

Related: [[client-server-session-split]] · [[session-store-format]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
