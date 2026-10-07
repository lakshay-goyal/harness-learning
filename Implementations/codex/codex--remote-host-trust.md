---
type: implementation
harness: codex
concept: remote-host-trust
commit: 622e9e3696
files: [codex-rs/websocket-auth/src/lib.rs:1, codex-rs/websocket-auth/src/lib.rs:26, codex-rs/app-server-transport/src/transport/websocket.rs:129, codex-rs/app-server-transport/src/transport/unix_socket.rs:31, codex-rs/uds/src/lib.rs:116, codex-rs/uds/src/daemon_directory.rs:23]
---
[[remote-host-trust]] in [[codex]].

## Mechanism
- **Transport-layer auth for WebSocket upgrades** (shared by app-server and exec-server): "Shared authentication for incoming app-server and exec-server WebSocket upgrades. Policies retain token digests or JWT verification secrets; callers choose when auth is required." (`codex-rs/websocket-auth/src/lib.rs:1-2`). Modes `--ws-auth capability-token` (SHA-256 digest, `constant_time_eq_32` compare) and `--ws-auth signed-bearer-token` (JWT via `jsonwebtoken`) (`codex-rs/websocket-auth/src/lib.rs:62-80`); `DEFAULT_MAX_CLOCK_SKEW_SECONDS = 30`, `MIN_SIGNED_BEARER_SECRET_BYTES = 32`, error text "invalid authorization header" (`:26-28`).
- **Refuse unauthenticated non-loopback listeners**: "refusing to start non-loopback websocket listener {bind_address} without auth; configure `--ws-auth capability-token` or `--ws-auth signed-bearer-token`" (`codex-rs/app-server-transport/src/transport/websocket.rs:129-142`).
- **Local sockets by filesystem permissions**: control socket mode `0o600`, directory `0o700` (`codex-rs/app-server-transport/src/transport/unix_socket.rs:31`, `:56`); `uds` socket dir `0o700` enforced/verified (`codex-rs/uds/src/lib.rs:116-149`; `codex-rs/uds/src/daemon_directory.rs:23-31` rejects dirs whose mode ≠ 0700).
- **Execution-host vs UI-host**: TUI-side filesystem precondition for `/init` moved into the prompt so it is checked on the execution host (`8e69d29521`; [[client-side-check-wrong-host]]).
- **Remote env trust**: PID-namespace inheritance only via trusted startup flag, never repo config/env (`codex-rs/linux-sandbox/README.md:96-108`); Noise relay auth tokens kept out of model-reachable children (`89e297729e`, `codex-rs/protocol/src/shell_environment.rs:13-21`).
- **Error boundary**: error diagnostics carry structure (category/position/size), not payloads or URLs (`977193486d`, `5a0d0929e2`) — see [[error-diagnostics-echo-payload]].
- No SSH layer: codex does not drive `ssh` itself (no known_hosts / host-key handling found by grep at `622e9e3696`, unverified beyond grep).

## Constants
| name | value | path:line |
|---|---|---|
| WS auth clock skew | 30 s | `codex-rs/websocket-auth/src/lib.rs:26` |
| min signed-bearer secret | 32 B | `codex-rs/websocket-auth/src/lib.rs:27` |
| control socket mode | `0o600` (dir `0o700`) | `codex-rs/app-server-transport/src/transport/unix_socket.rs:31` |

## Evolution
- 2026-05-12 `51bfb5f3b1` "Restore app-server websocket listener with auth guard (#22404)".
- 2026-08-17 `89e297729e` Noise auth tokens out of children.
- 2026-09-23 `c44deff7b1` "Extract WebSocket authentication into `codex-websocket-auth` (#47447)".

## Versus pi
- [[pi--remote-host-trust]]: app-owned pinned SSH host keys, all forwarding disabled, 0600 Unix socket, relay bearer on upgrade, `internal_error` sanitization. Codex: same transport-auth principle (capability token / JWT on the upgrade, 0600 sockets, refuse unauthenticated remote binds) but no SSH host-key layer.
