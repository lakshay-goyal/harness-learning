---
type: concept
stage: architecture
tier: candidate
aliases: ["--mode rpc", "RpcClient", "extension UI subprotocol", "strict-jsonl-framing", "stdout-protocol-guard", "takeOverStdout", "attachJsonlLineReader", "print mode", "-p", opencode run, opencode acp, ACP (Agent Client Protocol), "--format json"]
harnesses: [pi, opencode]
---
Long-lived subprocess protocol (line-delimited JSON commands → responses + streamed events, plus a UI request/response sub-protocol) so non-JS hosts (IDEs, GUIs, other agents) can drive the harness.

## Why
- Embedding without a language binding; isolation of the agent process from the host.
- Protocol integrity is fragile: generic line readers split on Unicode line separators and any stray stdout write corrupts the stream ([[headless-protocol-stream-corruption]]).
- Plugin dialogs (confirm/select) must still work when no terminal exists — needs a UI sub-protocol.
- Clients need to know when work is *finished*, not just accepted (pi: `prompt` response = accepted; wait for `agent_settled`, see [[run-settlement]]).

## Design space
- **Transport**: stdio JSONL (pi) vs socket/HTTP server vs LSP-style Content-Length framing vs CBOR length-prefix (pi experimental, [[client-server-session-split]]).
- **Framing**: Node `readline` (splits on U+2028/2029, rejected by pi) vs strict LF-only splitter (pi).
- **stdout ownership**: shared vs exclusive with global redirect of stray writes to stderr (pi `takeOverStdout`).
- **Correlation**: optional `id` on commands (pi) vs mandatory.
- **Ack semantics**: response on completion vs on acceptance with disposition (`started|queued|handled`) + lifecycle events (pi).
- **UI**: no UI (print/json) vs forwarded dialog primitives with timeouts and stubbed rich components (pi RPC).
- **Lifecycle**: close stdin = orderly shutdown (pi).
- **One-shot variants**: print (final text, exit code) and JSON event stream ([[agent-event-stream]]).
- **Headless entry points as HTTP clients of the harness's own server** (opencode `run`, ACP).
- **Editor protocol**: ACP over stdio NDJSON, permission prompts mapped to `requestPermission`, fail closed when unsupported (opencode).
- **Permission responder**: auto-reject every ask vs auto-approve non-denied (`--auto`) (opencode `run`); none (opencode GitHub run, [[approval-wait-without-responder]]).

## Implementations
- [[pi--headless-rpc-mode|pi]] — `--mode rpc`: 33 typed commands, strict LF JSONL, stdout takeover with ENOBUFS/EAGAIN retry, extension-UI subprotocol, TS `RpcClient`.
- [[opencode--headless-rpc-mode|opencode]] — `opencode run --format json` (auto-reject/`--auto` responder) and `opencode acp` (ACP NDJSON bridge to a local server); no bidirectional JSONL command protocol.

## Failures
- [[headless-protocol-stream-corruption]]
- [[quadratic-event-stream-output]]
- [[approval-wait-without-responder]]

## Tradeoffs
- [[client-server-vs-single-process]]

## Related
[[agent-event-stream]] · [[sdk-embedding]] · [[extension-ui-primitives]] · [[client-server-session-split]] · [[steering-queue]] · [[follow-up-queue]] · [[run-settlement]] · [[subagent-as-subprocess]] · [[ci-agent-integration]] · [[permission-ruleset]]
