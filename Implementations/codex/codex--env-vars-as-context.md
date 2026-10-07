---
type: implementation
harness: codex
concept: env-vars-as-context
commit: 622e9e3696
files: [codex-rs/core/src/exec_env.rs:14, codex-rs/core/src/exec_env.rs:18, codex-rs/core/src/exec_env.rs:30, codex-rs/core/src/exec_env.rs:39, codex-rs/core/src/exec_env.rs:54, codex-rs/protocol/src/shell_environment.rs:6, codex-rs/core/src/spawn.rs:21, codex-rs/core/src/spawn.rs:26, codex-rs/core/src/spawn.rs:92]
---
[[env-vars-as-context]] in [[codex]].

## Mechanism
- Command environment is built from the configured `ShellEnvironmentPolicy` and passed after `env_clear()` "to ensure no unintended variables are leaked to the spawned process" (`create_env`, `codex-rs/core/src/exec_env.rs:20-37`); "`CODEX_THREAD_ID` is injected when a thread id is provided, even when `include_only` is set."
- Injected variables:
  - `CODEX_THREAD_ID` (`codex-rs/protocol/src/shell_environment.rs:7`) — "so that the agent (and skills) can refer to the current thread / session ID" (`66b196a725` body).
  - `CODEX_SESSION_ID` shared root-session identity + `CODEX_VERSION` harness version (`inject_session_env`, `codex-rs/core/src/exec_env.rs:39-48`; `codex-rs/protocol/src/shell_environment.rs:6`; `codex-rs/core/src/exec_env.rs:14`).
  - `CODEX_PERMISSION_PROFILE` selected named permission profile, applied after the policy so the runtime value wins (`inject_permission_profile_env`, `codex-rs/core/src/exec_env.rs:18`, `:50-60`).
  - `CODEX_SANDBOX` ("seatbelt" on macOS when sandboxed) and `CODEX_SANDBOX_NETWORK_DISABLED=1` when network is sandboxed (`codex-rs/core/src/spawn.rs:21-26`, `:92`) — used e.g. by tests/clients to skip network (`codex-rs/lmstudio/src/client.rs:222`).
- Windows: case-insensitive removal of inherited duplicates before insert (`codex-rs/core/src/exec_env.rs:41-43`, `:58-59`).
- No prompt pointer: the base prompt does not mention these variables (no hit in `codex-rs/protocol/src/prompts/`); model-facing facts (date, cwd, shell, permissions) instead arrive in `<environment_context>` / world-state diffs → [[codex--world-state-diff-injection]].

## Evolution
- 2025-05-09 `fde48aaa0d` "feat: experimental env var: CODEX_SANDBOX_NETWORK_DISABLED (#879)".
- 2026-02-03 `66b196a725` "Inject CODEX_THREAD_ID into the terminal environment (#10096)".
- 2026-06-25 `c65cfeab14` "core: expose permission profile to shell tools (#29941)".
- 2026-08-10 `97729885d4` "Expose the session ID to shell commands (#37848)".
- 2026-09-02 `9bb1ea035f` "Expose the Codex version to commands and turn metadata (#42395)".

## Versus pi
- [[pi--env-vars-as-context]]: 5 `PI_*` vars (model, provider, session id/file, reasoning level) plus a softened prompt pointer ([[imperative-guideline-over-compliance]]). codex exposes identity/sandbox facts but no model name, and never points the model at them — avoiding the over-compliance pi hit.
