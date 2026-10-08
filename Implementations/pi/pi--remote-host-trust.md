---
type: implementation
harness: pi
concept: remote-host-trust
commit: b30a6dd77
files: [packages/env/src/ssh.ts:59, packages/env/src/ssh.ts:77, packages/env/src/ssh.ts:177, packages/env/src/ssh.ts:209, packages/env/daemon/src/main.rs:438, packages/server/src/transports/unix/listener.ts:11, packages/server/src/server.ts:512, packages/coding-agent/src/experimental/radius-relay.ts:626, packages/protocol/README.md:39]
---
[[remote-host-trust]] in [[pi]].

## Mechanism
### pi-env SSH bootstrap (remote execution host)
- **`sshArguments`** (`packages/env/src/ssh.ts:77-120`): `-T -a -x`, `BatchMode=yes` (no prompts); `ClearAllForwardings=yes`, `ForwardAgent=no`, `ForwardX11=no`; `ControlMaster=no`, `ControlPath=none` (no shared connections); `RemoteCommand=none`, `PermitLocalCommand=no`; `SendEnv=-*` (no locale/env forwarding); `ServerAliveInterval=15`; `StrictHostKeyChecking=yes`, `UserKnownHostsFile=<app file>`, `GlobalKnownHostsFile=none`, `HashKnownHosts=no`, `HostKeyAlias=<fixed alias>`; `IdentitiesOnly=yes` with an identity file; `--` before host. Port validated 1–65535.
- **Argument-injection guards**: host/user/alias must be non-empty, not start with `-`, no whitespace/control chars (`checkField`, `ssh.ts:59-64`); config paths quoted, `%` doubled to block token expansion (`configPath`, `:66-70`).
- **Explicit TOFU**: `scanHostKey` connects once with `accept-new` into a *temporary* known_hosts (`mkdtemp pi-env-hostkey-`) and returns key lines + `ssh-keygen -lf` fingerprints; doc: "Showing a fingerprint is not authentication: compare it with one obtained out of band before accepting it." (`ssh.ts:177-207`).
- **`acceptHostKey`**: only plain `alias type key` lines with type `ssh-*`, `ecdsa-sha2-*`, `sk-*` (`parseHostKeyLine`, `:210-220`); a **changed** key of an existing type throws `HostKeyChangedError` — "forget the old key first" (`:258-275`); `forgetHostKey` removes (`:282`). known_hosts rewrites serialized per file, temp file `wx` mode 0600 then `rename` (atomic) (`:224-251`).
- **Eager vs lazy**: `connectSsh()` detects/deploys eagerly, rejects `HostKeyUnknownError` for untrusted keys; `sshConnection()` defers to first op, failures (incl. untrusted key) become that op's `spawn_error`, next op retries (`:504-530`).
- Platform probe falls back to PowerShell on any failure *except* host-key errors (`:329-337`; `4bf5a6bc5`).
- Sync token `PI-ENV <token>`: daemon's first stdout line, used by the client only to discard shell-startup noise before frames (`packages/env/docs/protocol.md:5-9`; `packages/env/daemon/src/main.rs:438-443`; `packages/env/src/connection.ts:329-340`) — a sync marker, not authentication (inferred: no verification beyond `indexOf`); trust anchor = SSH.
- Remote binary integrity: content-addressed + hash-verified before every start ([[supply-chain-pinning]]).
### Experimental session server / relay (transport-layer auth)
- Protocol has no auth: "Peer authentication and authenticated service contexts are not implemented by the experimental transport" (`packages/protocol/README.md:39`); framed `auth {token}` removed in `305c014dc` (#7551) — transports must hand over authorized connections — "Supplies established byte connections after any required transport authentication" (`packages/server/src/listener.ts:3-8`).
- **Unix socket**: default mode `0o600` (`packages/server/src/transports/unix/listener.ts:11`), parent dir `0o700` (`:53`); atomic bind: listen on private `bind-<sha256(path)[:8]>`, `lstat` to verify socket + capture dev/ino, `link()` to public path, chmod, unlink private (`:52-80,294-297`); stale sockets removed only if probe (1 s) fails and inode unchanged (`:299-336`). Coordinator socket also `chmod 0o600` (`packages/coding-agent/src/experimental/coordinator.ts:570-572`). Security = filesystem permissions.
- **Server↔worker**: env `PI_SESSION_WORKER_CONTROL_TOKEN`, `PI_SESSION_WORKER_SESSION_KEY_BASE64`; each control message carries `token`/`sessionKey` (`packages/coding-agent/src/experimental/session-worker.ts:64-67,138-175,536-539`).
- **Radius relay** (remote access): host dials out `wss://<gateway>/v1/session-relays/<serverId>/connect`, subprotocol `pi-session-relay.host.v1` / `.client.v1` (`packages/coding-agent/src/experimental/radius-relay.ts:7-8,618-624`); `Authorization: Bearer <token>` on the WebSocket upgrade (`:626-646`); client fails unless server selected exactly the requested subprotocol (`:662-667`); credentials re-resolved per connection attempt, OAuth refreshed if <5 min validity, `PI_OFFLINE` disables (`packages/coding-agent/src/experimental/radius-auth.ts:16,36-50`)
. No E2E crypto above TLS — gateway sees the byte stream (grep finds no crypto in `radius-relay.ts`).
- **Error sanitization**: `toProtocolError` passes `ServerError`/`RemoteServiceError` codes, maps `ProtocolValidationError` → `invalid_request`, everything else → `{code:"internal_error", message: INTERNAL_SERVER_ERROR_MESSAGE}` and reports the cause locally (`packages/server/src/server.ts:512-521`; `dab70b2cd`, #7644). Codec error messages truncated to 500 chars (`packages/protocol/src/codec.ts:34-37`). Chord wire error taxonomy is a fixed list (`packages/chord/src/services/errors.ts:1-10`).
- Client-side: `assertAccess() {}` is a no-op (`packages/coding-agent/src/experimental/client-runtime.ts:158-160`); clients may not pass plugin package paths through Radius (`:66-68`).

## Constants
| name | value | path:line |
|---|---|---|
| `ServerAliveInterval` | 15 | packages/env/src/ssh.ts:103 |
| `SSH_TIMEOUT_MS` / `UPLOAD_TIMEOUT_MS` | 60 s / 300 s | packages/env/src/ssh.ts:122-124 |
| Unix socket mode | `0o600` | packages/server/src/transports/unix/listener.ts:11 |
| `SOCKET_PROBE_TIMEOUT_MS` | 1 s | packages/server/src/transports/unix/listener.ts:15 |
| `DEFAULT_HANDSHAKE_TIMEOUT_MS` | 5 s | packages/server/src/server.ts:42 |
| Radius OAuth min validity | 5 min | packages/coding-agent/src/experimental/radius-auth.ts:44-50 |
| relay close codes | 4000 protocol / 4001 transport | packages/coding-agent/src/experimental/radius-relay.ts:21-25 |

## Evolution
- 2026-07-14 `961fa6c14` Radius gateway support added to `pi-ai` (`packages/coding-agent/src/core/radius.ts` created) — the credential path the later relay reuses.
- 2026-07-30 `56eb685b6` pi-protocol v2 with domain verbs; 2026-08-03 `305c014dc` framed bearer auth removed, version reset to 1, "Require transports to establish authorized connections" (#7551).
- 2026-08-05 `dab70b2cd` sanitize service failures to `internal_error`/`not_implemented` (#7644).
- 2026-08-14 `9fcaac9b1` bounded Unix socket paths; `80b6797c9` canonical UUIDv4 server ids.
- 2026-08-28 `1d0d110ab` Radius relay with bearer auth; `35c49350f` close-code fix.
- 2026-10-05 `b78e6a908` SSH bootstrap: strict host keys under alias; `ed94330a2` hardened: all forwarding/control/RemoteCommand off, `GlobalKnownHostsFile=none`, parsed host-key lines, changed keys refused; `6fa21f2a3` 60/300/60 s timeouts.

## Evidence commits
`305c014dc`, `dab70b2cd`, `1d0d110ab`, `b78e6a908`, `ed94330a2`, `4bf5a6bc5`, `6fa21f2a3`

## Quirks
- Gateway-side authorization of `serverId` vs token is closed source (unverified); an org member's access to another's server unknown.
- Lazy mode turns an untrusted host key into an ordinary per-op `spawn_error` that retries every op.

## Failures
- [[internal-errors-leak-over-wire]] · [[auth-at-message-layer]] · [[remote-binary-trusted-by-version-name]]
