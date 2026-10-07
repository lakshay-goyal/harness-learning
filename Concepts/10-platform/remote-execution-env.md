---
type: concept
stage: architecture
tier: candidate
aliases: ["pi-env", "RemoteExecutionEnv", "Connection", "RemoteWatcher", "connectSsh", "sshConnection", "pi-env serve", "remote-execution-daemon", "file-watch-coverage-contract", "FileSystem.watch", "NodeFileWatcher", codex exec-server, codex-exec-server, ExecServerClient, TurnEnvironment, environment_id, ExecutorFileSystem]
harnesses: [pi, codex]
---
Run the agent's tools on another machine: a small native daemon deployed over SSH performs filesystem, exec and watch operations for a local agent over a framed stdio protocol, while the agent, its storage and credentials stay local.

## Why
- Credentials and session state never leave the local machine; the remote box only sees tool I/O ([[tool-only-isolation]]).
- Tool semantics must be identical locally and remotely or the model's learned behavior breaks — differential/conformance testing is mandatory ([[remote-errno-parity-drift]]).
- Remote links are slow and lossy: bulk output must not starve pings/cancels ([[bulk-output-starves-control-messages]]), reconnects must not misroute handles ([[stale-handle-reaches-new-daemon]]), liveness must count bytes not frames ([[liveness-timeout-counts-frames-not-bytes]]).
- Bootstrap over heterogeneous OSes hangs or misdetects without timeouts and in-band framing ([[remote-bootstrap-hang]], [[remote-platform-misdetection]]).
- File watching has platform-specific gaps ([[file-watch-platform-gaps]]).

## Design space
- **Remote mechanism**: run commands over plain `ssh` per tool call (pi `ssh.ts` example) vs persistent daemon with framed protocol (pi-env) vs full remote agent (whole-process isolation, [[tool-only-isolation]]).
- **Data framing**: JSON+base64 vs JSON header + raw byte payload (pi-env).
- **Output transfer**: stream everything vs apply caller's retained tail window at the source and report skipped counts (pi-env, see [[shell-execution]]).
- **Liveness**: TCP keepalive vs bidirectional pings + dead-man switch killing all process groups on silence (pi-env, [[process-tree-kill]]).
- **Deploy**: preinstalled vs content-addressed upload verified by SHA-256 before every start (pi-env, [[supply-chain-pinning]]).
- **Host trust**: user known_hosts vs app-owned pinned known_hosts with explicit fingerprint acceptance ([[remote-host-trust]]).
- **Sandboxing**: none (pi-env: full SSH-user rights) vs path allowlist/jail.
- **Watching**: client-side polling over the link (pi-env initial, replaced) vs native watcher next to the files with coverage contract (pi-env now).
- **Remote mechanism (codex)**: persistent JSON-RPC exec/fs server over websocket, reached directly or via environment registry + Noise relay rendezvous; forward mode passes payloads unchanged, no replay ✔ codex.
- **Multi-environment turns**: tools take `environment_id`; patches carry `*** Environment ID:` ✔ codex.
- **Lifecycle**: websocket close kills that client's processes; sequential requests unless `--concurrent-requests` ✔ codex.

## Implementations
- [[pi--remote-execution-env|pi]] — `@earendil-works/pi-env`: Rust daemon (16-thread pool, control/bulk queues, windowed exec, native watch) + TS `RemoteExecutionEnv` implementing pi-durable's `ExecutionEnv`; hardened SSH bootstrap. No consumer in the monorepo at HEAD.
- [[codex--remote-execution-env|codex]] — `codex exec-server`: JSON-RPC process + filesystem server over websocket (direct, registry + Noise relay, or forwarder); tools run against a `TurnEnvironment` with `environment_id` in multi-environment turns.

## Failures
- [[bulk-output-starves-control-messages]]
- [[stale-handle-reaches-new-daemon]]
- [[liveness-timeout-counts-frames-not-bytes]]
- [[remote-bootstrap-hang]]
- [[remote-platform-misdetection]]
- [[remote-errno-parity-drift]]
- [[file-watch-platform-gaps]]
- [[remote-binary-trusted-by-version-name]]
- [[conformance-suite-platform-timing]] (10-platform) — Shared env conformance cases timed out under the runner's default timeout (watch cases wait for polling…

## Related
[[pluggable-tool-backends]] · [[tool-only-isolation]] · [[remote-host-trust]] · [[process-tree-kill]] · [[supply-chain-pinning]] · [[shell-execution]] · [[file-read-tool]] · [[tool-output-spill]] · [[durable-execution]] · [[client-server-session-split]] · [[harness-evals]] · [[no-sandbox]]
