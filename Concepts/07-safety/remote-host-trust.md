---
type: concept
stage: permissions
tier: candidate
aliases: [sshArguments, scanHostKey, acceptHostKey, forgetHostKey, HostKeyChangedError, HostKeyUnknownError, StrictHostKeyChecking, Unix socket 0600, Bearer relay token, internal_error, pinned-host-key-ssh, transport-layer-auth, error-sanitization-at-boundary]
harnesses: [pi]
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
- **Where auth lives**: framed auth message (pi protocol v2, removed) vs transport (0600 Unix socket; relay `Authorization: Bearer` on WebSocket upgrade) — pi now.
- **Error boundary**: pass-through vs allow-listed codes, everything else `internal_error` (pi server).
- **E2E encryption** above TLS for relays — absent in pi (gateway sees traffic).

## Implementations
- [[pi--remote-host-trust|pi]] — pi-env `sshArguments` (BatchMode, ClearAllForwardings, StrictHostKeyChecking=yes, app known_hosts), explicit `scanHostKey`/`acceptHostKey`, `HostKeyChangedError`; server Unix socket 0600 + atomic bind; Radius relay bearer; `internal_error` sanitization.

## Failures
- [[internal-errors-leak-over-wire]]
- [[auth-at-message-layer]]
- [[remote-binary-trusted-by-version-name]]

## Related
[[remote-execution-env]] · [[client-server-session-split]] · [[tool-only-isolation]] · [[supply-chain-pinning]] · [[process-tree-kill]]
