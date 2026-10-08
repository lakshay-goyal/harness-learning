---
type: concept
stage: permissions
tier: must-have
aliases: [sshArguments, scanHostKey, acceptHostKey, forgetHostKey, HostKeyChangedError, HostKeyUnknownError, StrictHostKeyChecking, Unix socket 0600, Bearer relay token, internal_error, pinned-host-key-ssh, transport-layer-auth, error-sanitization-at-boundary, OPENCODE_SERVER_PASSWORD, auth_token query, PTY connect ticket, codex-websocket-auth, --ws-auth, capability-token, signed-bearer-token]
harnesses: [pi, opencode, codex]
---
When the agent reaches across a process/network boundary (remote execution host, session server, relay), authenticate at the transport (app-owned pinned host keys, filesystem-permissioned sockets, bearer tokens on the upgrade), disable every implicit channel (agent/X11/port forwarding, connection sharing, env forwarding), and let only stable error codes — never internals — cross back.

## Why
- Remote exec over the user's normal SSH config inherits agent forwarding, ControlMaster sockets, RemoteCommand and global known_hosts: an attacker-controlled host or config hijacks the user's credentials.
- "Showing a fingerprint is not authentication": TOFU must be explicit and a *changed* key must never auto-accept.
- In-band auth messages couple auth to protocol versions ([[auth-at-message-layer]]); leaked stack traces reveal internals to remote clients ([[internal-errors-leak-over-wire]]).

## Design space
- **Use user's ssh config + global known_hosts** vs **app-owned known_hosts under a fixed `HostKeyAlias`** with `GlobalKnownHostsFile=none` (pi-env).
- **Host-key acceptance**: silent `accept-new` vs explicit scan → out-of-band fingerprint comparison → accept; changed key ⇒ error until explicitly forgotten (pi-env).
- **Implicit channels**: default vs all forwarding/control/RemoteCommand/LocalCommand/SendEnv disabled (pi-env).
- **Argument injection**: host/user/alias fields validated (no leading `-`, whitespace, control chars) and `--` before host.
- **Where auth lives**: framed auth message (pi protocol v2, removed) vs transport (0600 Unix socket; relay `Authorization: Bearer` on WebSocket upgrade) — ✔ pi now; ✔ codex (capability token digest or signed JWT on the WebSocket upgrade, 0600/0700 Unix sockets).
- **Unauthenticated remote listener**: allowed vs refused (✔ codex refuses non-loopback binds without `--ws-auth`).
- **SSH host-key layer**: pi-env only; codex has no SSH driver (unverified beyond grep).
- **Execution-host preconditions**: checked on the UI host vs on the execution host (codex moved `/init`'s AGENTS.md check into the prompt — [[client-side-check-wrong-host]]).
- **Error boundary**: pass-through vs allow-listed codes, everything else `internal_error` (pi server).
- **E2E encryption** above TLS for relays — absent in pi (gateway sees traffic).
- **Unauthenticated by default**: loopback bind + printed warning, HTTP Basic only when a password is set (opencode `serve`); declared the user's responsibility (`SECURITY.md:21-29`).
- **Browser clients**: CORS allowlist incl. the vendor web app origin; WebSockets checked by Origin/Host + connect tickets (opencode).

## Implementations
- [[pi--remote-host-trust|pi]] — pi-env `sshArguments` (BatchMode, ClearAllForwardings, StrictHostKeyChecking=yes, app known_hosts), explicit `scanHostKey`/`acceptHostKey`, `HostKeyChangedError`; server Unix socket 0600 + atomic bind; Radius relay bearer; `internal_error` sanitization.
- [[codex--remote-host-trust|codex]] — `codex-websocket-auth` (capability-token / signed-bearer-token, 30 s skew, ≥ 32 B secret) for app-server and exec-server upgrades; refuses unauthenticated non-loopback listeners; 0600 control socket.
- [[opencode--remote-host-trust|opencode]] — optional Basic auth (`OPENCODE_SERVER_PASSWORD`), `127.0.0.1` default, CORS allowlist, PTY tickets; remote workspaces proxied with no host pinning.

## Failures
- [[internal-errors-leak-over-wire]]
- [[auth-at-message-layer]]
- [[remote-binary-trusted-by-version-name]]
- [[client-side-check-wrong-host]] · [[error-diagnostics-echo-payload]]

## Related
[[remote-execution-env]] · [[client-server-session-split]] · [[tool-only-isolation]] · [[supply-chain-pinning]] · [[process-tree-kill]] · [[location-scoped-runtime]]
