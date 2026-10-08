---
type: implementation
harness: codex
concept: headless-rpc-mode
commit: 622e9e3696
files: [codex-rs/exec/src/cli.rs:18, codex-rs/exec/src/lib.rs:343, codex-rs/exec/src/lib.rs:587, codex-rs/exec/src/lib.rs:1010, codex-rs/exec/src/exec_events.rs:13, codex-rs/app-server/src/main.rs:30]
---
[[headless-rpc-mode]] in [[codex]].

Two headless surfaces: one-shot `codex exec` (optionally JSONL events) and the long-lived `codex app-server --listen stdio://` protocol ([[codex--client-server-session-split|client-server-session-split]]).

## Mechanism
- **`codex exec`** runs on the in-process app-server client (`InProcessAppServerClient::start`, `codex-rs/exec/src/lib.rs` ~line 1020; `da3689f0ef` 2026-03-08).
- **Headless safety defaults**: approval policy forced `Never` ("Default to never ask for approvals in headless mode") unless the resolved reviewer is auto-review (`codex-rs/exec/src/lib.rs:587-592`) → [[approval-policy-modes]], [[llm-approval-reviewer]]; refuses to run outside a git repo unless `--skip-git-repo-check` or `--dangerously-bypass-approvals-and-sandbox` ("Not inside a trusted directory and --skip-git-repo-check was not specified.", `codex-rs/exec/src/lib.rs:1010-1018`); bypass flag maps to `SandboxMode::DangerFullAccess` (`codex-rs/exec/src/lib.rs:343-347`).
- **Flags** (`codex-rs/exec/src/cli.rs:18-330`): `--json` (alias `--experimental-json`), `--output-schema FILE` (final message constrained by JSON schema; also on resume since `af6ffb6ebb` 2026-05-18), `-o/--output-last-message FILE`, `--ephemeral`, `--ignore-user-config`, `--ignore-rules`, `--strict-config`, `--thread-source`, `--image`, global `--worktree` (`codex-rs/exec/src/cli.rs:169`) → [[git-worktree-isolation]]. Subcommands `resume` (`--last`, `--all`), `fork`, `review` (`--uncommitted`, `--base`, `--commit`, `--title`).
- **Event schema**: dot-named JSONL `ThreadEvent`s (`codex-rs/exec/src/exec_events.rs:13-36`) → [[codex--agent-event-stream|agent-event-stream]].
- **Long-lived protocol**: `codex app-server` over stdio/unix/ws with typed requests, server→client approval requests and capability negotiation (`codex-rs/app-server/src/main.rs:30-75`).
- **Framing hazard elsewhere**: JS line terminators U+2028/2029 broke JSONL framing in the js_repl kernel → [[line-separator-breaks-jsonl-framing]].

## Evolution
- 2025-09-30 `d9dbf48828` app-server split from `codex mcp`.
- 2026-03-08 `da3689f0ef` exec on in-process app server.
- 2026-05-18 `af6ffb6ebb` `--output-schema` on resume.
- 2026-09-05 `531f3836a1` `codex mcp-server` (Codex as an MCP server) removed.
- Interrupt RPC on a finished turn hung → [[interrupt-rpc-hangs-on-finished-turn]].

## Versus pi
- [[pi--headless-rpc-mode]]: pi `--mode rpc` = 33 LF-JSONL commands + extension-UI subprotocol on one stdio process; codex splits one-shot (`exec`, JSONL events, schema-constrained output) from the full RPC (app-server, ~190 methods) and makes approvals server→client requests rather than a UI subprotocol.
- Headless default differs sharply: pi runs YOLO in every mode; codex exec forces approval `never` *with* the sandbox still on and requires a git repo.
