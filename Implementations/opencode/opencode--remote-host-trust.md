---
type: implementation
harness: opencode
concept: remote-host-trust
commit: ecc4916b5a
files: [packages/opencode/src/server/auth.ts:18-32, packages/server/src/auth.ts:31-56, packages/opencode/src/server/routes/instance/httpapi/middleware/authorization.ts:12, packages/server/src/cors.ts:3-31, packages/opencode/src/server/routes/instance/httpapi/handlers/pty.ts:28-30, packages/opencode/src/cli/network.ts:15-73, packages/opencode/src/cli/cmd/serve.ts:16, SECURITY.md:21-29]
---
[[remote-host-trust]] in [[opencode]].

## Mechanism
### Legacy runtime (HTTP server)
- **Auth at the transport**: optional HTTP Basic. Password from `OPENCODE_SERVER_PASSWORD`, username `OPENCODE_SERVER_USERNAME ?? "opencode"` (`packages/opencode/src/server/auth.ts:18-32`). Without a password the server runs open and prints "Warning: OPENCODE_SERVER_PASSWORD is not set; server is unsecured." (`packages/opencode/src/cli/cmd/serve.ts:16`).
- Browser `EventSource`/WebSocket clients can pass credentials as `?auth_token=` (`packages/opencode/src/server/routes/instance/httpapi/middleware/authorization.ts:12`).
- **Binding**: default `127.0.0.1`; `--mdns` without an explicit hostname binds `0.0.0.0` (`packages/opencode/src/cli/network.ts:15`, `packages/opencode/src/cli/network.ts:65-73`). The TUI uses no TCP port at all unless network flags are given ([[client-server-session-split]]).
- **CORS allowlist** (`packages/server/src/cors.ts:3-31`): `http://localhost:*`, `http://127.0.0.1:*`, `oc://renderer`, tauri origins, `https://*.opencode.ai`, plus configured origins; same-host requests and requests without `Origin` pass.
- **PTY WebSockets** bypass CORS, so the handler checks Origin/Host (`packages/opencode/src/server/routes/instance/httpapi/handlers/pty.ts:28-30`) and requires connect tickets (`7bc26dafae`).
- **Remote workspaces**: requests for a remote workspace are HTTP-proxied to the remote opencode server with `x-opencode-directory`/`x-opencode-workspace` stripped (`packages/opencode/src/server/proxy-util.ts:17-18`); trust in the remote host is whatever the adapter provides (no host-key pinning in opencode itself).
- **Stance**: "Server mode is opt-in only … It is the end user's responsibility to secure the server - any functionality it provides is not a vulnerability" (`SECURITY.md:21-29`).

### v2 runtime
- `packages/server` re-implements the same Basic check (`packages/server/src/auth.ts:31-56`) and shares the CORS helper.

## Constants
| name | value | path:line |
|---|---|---|
| default host | `127.0.0.1` | `packages/opencode/src/cli/network.ts:15` |
| default username | `opencode` | `packages/opencode/src/server/auth.ts:19` |

## Evolution
- 2025-12-26 `505068d5a6` reverted optional mDNS; mDNS present again at HEAD (`packages/opencode/src/server/mdns.ts`).
- 2026-01-12 `1954c1255e` password authentication + server hardening.
- 2026-05-03 `7bc26dafae` PTY WebSocket auth tickets.

## Quirks / drift
- Password compared with plain `===` (`packages/opencode/src/server/auth.ts:31-32`; `packages/server/src/auth.ts:47-48`), not constant-time (observed, minor).
- Any `*.opencode.ai` page may call an unauthenticated local server from the browser (allowlisted origin) — by design for the hosted web app.

pi contrast: pi authenticates with 0600 Unix sockets and relay bearer tokens, and pins remote SSH host keys ([[pi--remote-host-trust|pi]]).
