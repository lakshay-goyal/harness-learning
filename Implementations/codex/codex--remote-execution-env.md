---
type: implementation
harness: codex
concept: remote-execution-env
commit: 622e9e3696
files: [codex-rs/exec-server/README.md:1, codex-rs/exec-server/README.md:36, codex-rs/exec-server/README.md:166, codex-rs/exec-server/README.md:430, codex-rs/exec-server/src/client.rs:167, codex-rs/core/src/tools/runtimes/apply_patch.rs:1, codex-rs/core/src/tools/spec_plan.rs:1168]
---
[[remote-execution-env]] in [[codex]].

`codex exec-server`: a small JSON-RPC process + filesystem server; the agent (and its credentials/state) stays where codex runs, while tools execute against a `TurnEnvironment` that may be remote.

## Mechanism
- "`codex-exec-server` is the library backing `codex exec-server`, a small JSON-RPC server for spawning and controlling subprocesses through `codex-utils-pty`" — CLI entrypoint, Rust `ExecServerClient`, shared protocol module (`codex-rs/exec-server/README.md:1-16`). Own envelope `codex-exec-server-protocol`.
- **Transports**: `ws://IP:PORT` (default); `--remote URL --environment-id ID [--name NAME]` registers with an environment registry and reconnects to a service-provided rendezvous websocket over a Noise relay; `forward --connect ws://HOST:PORT …` opens an independent websocket per authenticated harness stream and passes payloads unchanged — "The forwarder does not replay requests or persist execution state, so recovery is limited by the destination's session and process-output retention" (`codex-rs/exec-server/README.md:17-60`).
- **Auth**: opt-in websocket auth (capability token sha256, token files, signed bearer); tokens require `wss://` or loopback, redacted in diagnostics, reused on reconnect without refresh; registry auth via ChatGPT sign-in, `CODEX_API_KEY`, Agent Identity JWT (`--use-agent-identity-auth`), or AWS SigV4 (`codex-rs/exec-server/README.md:28-60`).
- **Lifecycle**: `initialize` → response → `initialized` → process/fs RPCs; requests sequential by default (`--concurrent-requests <COUNT>`); any notification other than `initialized` gets an error with id `-1`; websocket close terminates that client's managed processes (`codex-rs/exec-server/README.md:166-182`).
- **API**: `process/start|read|write|terminate`, notifications `process/output|exited|closed`; filesystem `fs/readFile`, `fs/open|readBlock|writeBlock|close` (handles; `open.mode` read|replace), `fs/writeFile`, `fs/createDirectory`, `fs/getMetadata`, `fs/canonicalize`, `fs/readDirectory`, `fs/remove`, `fs/copy` (`codex-rs/exec-server/README.md:184-457`).
- **Harness side**: tools resolve a `TurnEnvironment` (local or remote); `apply_patch` reads/writes through `ExecutorFileSystem` with an explicit sandbox context (`codex-rs/core/src/tools/runtimes/apply_patch.rs:1-5`, `:177-181`); multi-environment turns add `environment_id` to exec_command/apply_patch/view_image (`codex-rs/core/src/tools/spec_plan.rs:1168`, `:1344-1347`, `:1358-1362`) and an `*** Environment ID:` line inside patches; env-bound tools hidden when no exec server (`a504d8f0fa` 2026-04-06) → [[pluggable-tool-backends]]. App-server `environment/add` with `authBearerToken`.
- Daemon recovery requires a single environment (`codex-rs/core/src/session/daemon_recovery.rs:9-40`).

## Constants
| name | value | path:line |
|---|---|---|
| exec-server connect / initialize timeouts | 10 s / 10 s | codex-rs/exec-server/src/client.rs:167-170 |
| default listen | `ws://IP:PORT` | codex-rs/exec-server/README.md:24 |
| request concurrency | sequential unless `--concurrent-requests` | codex-rs/exec-server/README.md:175-176 |

## Evolution
- 2025-11-18 `c1391b9f94` "exec-server (#6630)" (first, execve-wrapper oriented); deleted 2026-02-23 `38f84b6b29` (execve wrapper moved to `codex-shell-escalation`).
- 2026-03-18 `81996fcde6` exec-server reborn as remote execution server.
- 2026-04-06 `a504d8f0fa` disable env-bound tools when exec server is none.

## Versus pi
- [[pi--remote-execution-env]]: pi-env is a Rust daemon deployed over SSH with framed stdio, control/bulk queues and native watching, with no consumer in the monorepo; codex's exec-server is wired into the tool layer (`TurnEnvironment`, `environment_id`) and reaches hosts via websocket + Noise relay registry rather than SSH bootstrap.
