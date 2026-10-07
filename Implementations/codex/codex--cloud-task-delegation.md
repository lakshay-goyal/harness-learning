---
type: implementation
harness: codex
concept: cloud-task-delegation
commit: 622e9e3696
files: [codex-rs/cloud-tasks/src/cli.rs:17, codex-rs/cloud-tasks/src/cli.rs:52, codex-rs/cloud-tasks-client/src/api.rs:136, codex-rs/cloud-client/src/lib.rs:1, codex-rs/cloud-client/src/lib.rs:32]
---
[[cloud-task-delegation]] in [[codex]].

## Mechanism
- `codex apply` (crate `codex-rs/chatgpt`, `ApplyCommand`): "Applies the latest diff from a Codex agent task" locally (`codex-rs/chatgpt/src/apply_command.rs:14-22`).
- **CLI**: `codex cloud exec --env ENV_ID [--attempts 1..4] [--branch]`, `status`, `list` (1–20 per page, `--json`), `apply [--attempt N]`, `diff` (`codex-rs/cloud-tasks/src/cli.rs:17-120`); attempts validated "between 1 and 4" (`:52-61`). TUI task browser `codex-rs/cloud-tasks/src/app.rs` / `ui.rs`; environment auto-detect `env_detect.rs`.
- **Backend trait**: list_tasks, get_task_summary / diff / messages / text, list_sibling_attempts, apply_task_preflight, apply_task, create_task (`codex-rs/cloud-tasks-client/src/api.rs:136-175`); mock backend crate for tests (`cloud-tasks-mock-client`).
- **gRPC client** for a cloud ThreadService (`codex-rs/cloud-client/src/lib.rs:1-5`): "Requests are never retried: a timeout or disconnect can leave Resume admitted. Attach delivers live events only; dropping its stream detaches without interrupting the thread or answering approvals."

## Constants
| name | value | path:line |
|---|---|---|
| best-of-N attempts | 1..4 | `codex-rs/cloud-tasks/src/cli.rs:52-61` |
| list page size | 1–20 | `codex-rs/cloud-tasks/src/cli.rs:17-120` |
| gRPC request timeout | `REQUEST_TIMEOUT` = 150 s, no retry | `codex-rs/cloud-client/src/lib.rs:5`, `:32` |

## Versus pi
- pi has no hosted-task delegation (not in findings); pi's remote work goes through its own client/server split ([[client-server-session-split]]).
