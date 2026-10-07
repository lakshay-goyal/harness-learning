---
type: implementation
harness: codex
concept: sdk-embedding
commit: 622e9e3696
files: [sdk/typescript/src/exec.ts:1, sdk/typescript/src/codex.ts:1, sdk/python/src/openai_codex/client.py:215, sdk/python-runtime/README.md:1, codex-rs/app-server-client/src/lib.rs:1, codex-rs/thread-manager-sample/src/main.rs:1]
---
[[sdk-embedding]] in [[codex]].

No SDK links the agent in-process; both public SDKs are **subprocess wrappers**, with two different wire choices.

## Mechanism
- **TypeScript** `@openai/codex-sdk` (`sdk/typescript`): spawns `codex exec --experimental-json` and parses the JSONL ThreadEvent stream (`sdk/typescript/src/exec.ts:1`, `:94-145`); API Codex → Thread → `run` / `runStreamed` (`sdk/typescript/src/codex.ts`, `sdk/typescript/src/thread.ts`); `--output-schema` via temp file (`sdk/typescript/src/outputSchemaFile.ts`).
- **Python** `openai-codex` (`sdk/python`): "Synchronous typed JSON-RPC client for `codex app-server` over stdio", spawning `codex app-server --listen stdio://` (`sdk/python/src/openai_codex/client.py:215`, `:256-272`); generated protocol types (`sdk/python/src/openai_codex/generated`), approvals, goals, login helpers.
- **`sdk/python-runtime`** (`openai-codex-cli-bin`): wheel-only platform package pinning an exact Codex CLI binary "so the SDK can pin an exact Codex CLI version without checking platform binaries into the repo" (`sdk/python-runtime/README.md`).
- **Rust in-process**: `codex-rs/app-server-client` is the only in-process facade, used by TUI and exec (`codex-rs/app-server-client/src/lib.rs:1-17`); `codex-rs/thread-manager-sample/src/main.rs` is an example binary driving `codex_core_api` (ThreadManager, AuthManager…) directly — reference for embedding core without app-server.
- `codex-rs/core-api` (crate `codex-core-api`): "Public facade for thread management APIs built on `codex-core`" (`codex-rs/core-api/src/lib.rs:1`) — the crate `thread-manager-sample` embeds.
- Release: separate workflows for Python SDK/runtime and npm SDK (`.github/workflows/*`).

## Versus pi
- [[pi--sdk-embedding]]: pi's SDK is the in-process `createAgentSession()` with injectable services, and its own evals use it; codex keeps the agent behind a process boundary (exec JSONL or app-server JSON-RPC) and exposes in-process embedding only to Rust (`codex_core_api`).
