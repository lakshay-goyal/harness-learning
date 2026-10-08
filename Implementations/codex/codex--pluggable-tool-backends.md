---
type: implementation
harness: codex
concept: pluggable-tool-backends
commit: 622e9e3696
files: [codex-rs/file-system/src/lib.rs:639, codex-rs/core/src/tools/runtimes/apply_patch.rs:1-5, codex-rs/core/src/tools/runtimes/apply_patch.rs:170-181, codex-rs/core/src/tools/handlers/mod.rs:160, codex-rs/core/src/tools/spec_plan.rs:1154-1168, codex-rs/core/src/tools/spec_plan.rs:1344-1362, codex-rs/core/src/tools/handlers/apply_patch_spec.rs:10-16, codex-rs/exec-server/src/client.rs:167-171, codex-rs/exec-server/README.md]
---
[[pluggable-tool-backends]] in [[codex]] — tools resolve a `TurnEnvironment` (local or remote exec-server) and do all process/filesystem I/O through its API; a turn can have several environments and the model picks one per call.

## Mechanism
- `ExecutorFileSystem` trait (`codex-rs/file-system/src/lib.rs:639`) — filesystem accessor used by `apply_patch` (reads/writes with an explicit `FileSystemSandboxContext`) for local and remote environments (`codex-rs/core/src/tools/runtimes/apply_patch.rs:1-5,170-181`); `apply_patch` verification also reads through it (`codex-rs/apply-patch/src/invocation.rs:141-147`).
- `resolve_tool_environment(step_context, environment_id, unavailable_msg)` picks the environment for a call (`codex-rs/core/src/tools/handlers/mod.rs:160`); unavailable → e.g. "view_image is unavailable in this session" / "apply_patch environment selection is unavailable for this turn".
- Multi-environment turns: `environment_id` param added to `exec_command` / `apply_patch` / `view_image` (`codex-rs/core/src/tools/spec_plan.rs:1168,1344-1347,1358-1362`), described as "Environment id from <environment_context>. Omit to use the primary environment."; inside patches an `*** Environment ID: ` header line (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:10-16`) → [[patch-envelope-edit]]. Environments listed to the model in `<environment_context>` ([[world-state-diff-injection]]).
- Environment-bound tools hidden entirely when no environment is attached (`advertise_environment_tools`, `spec_plan.rs:1154-1156`; `a504d8f0fa` 2026-04-06 "Disable env-bound tools when exec server is none"); `wait_for_environment` tool under `Feature::DeferredExecutor` (off) for environments that attach later.
- **exec-server** (`codex-rs/exec-server`, reborn `81996fcde6` 2026-03-19 as remote execution server): "a small standalone JSON-RPC server for spawning and controlling subprocesses through `codex-utils-pty`" + filesystem handlers; the top-level `codex` binary owns hidden helper dispatch for sandboxed fs ops (`codex-rs/exec-server/README.md`). Client timeouts: connect 10 s, initialize 10 s, environment info 30 s, environment status 10 s, process event channel 256 (`codex-rs/exec-server/src/client.rs:167-171`) → [[remote-execution-env]].
- `PathUri` keeps environment-native paths ("without projecting them onto the app-server or exec-server host") → [[path-normalization]].
- Shell snapshots V1 are host-side; V2 (executor-side) under development ([[shell-environment-snapshot]]).
- Hooks as the user-level seam: PreToolUse may rewrite tool input; no per-tool operations interface for embedders (unlike pi) — unverified beyond `codex-rs/core/src/tools/registry.rs:626-678`.

## Constants
| name | value | path:line |
|---|---|---|
| exec-server connect / initialize timeout | 10 s / 10 s | `codex-rs/exec-server/src/client.rs:167-168` |
| environment info / status timeout | 30 s / 10 s | `codex-rs/exec-server/src/client.rs:169-170` |
| process event channel capacity | 256 | `codex-rs/exec-server/src/client.rs:171` |

## Evolution
- 2026-02-23 `38f84b6b29` exec-server v1 removed → 2026-03-19 `81996fcde6` reborn as remote execution server (stub + protocol docs).
- 2026-04-06 `a504d8f0fa` env-bound tools hidden without exec server.
- 2026-08-18 `681c82f497` snapshot scripts in `codex-shell-command` (prep for executor-side snapshots).

## Versus pi
pi: per-tool `*Operations` interfaces + `spawnHook` (coding-agent) and one `ExecutionEnv` with typed `Result`s (durable), remote Rust daemon ([[pi--pluggable-tool-backends]]). codex: one environment abstraction (filesystem + process via exec-server JSON-RPC), selectable per call by the **model** through `environment_id`.
