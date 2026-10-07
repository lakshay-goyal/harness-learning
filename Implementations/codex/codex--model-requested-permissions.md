---
type: implementation
harness: codex
concept: model-requested-permissions
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/shell_spec.rs:193, codex-rs/core/src/tools/handlers/shell_spec.rs:232, codex-rs/core/src/tools/handlers/request_permissions.rs:32, codex-rs/protocol/src/request_permissions.rs:12, codex-rs/core/src/tools/orchestrator.rs:236, codex-rs/prompts/src/permissions_instructions.rs:29]
---
[[model-requested-permissions]] in [[codex]].

## Mechanism
- Tool `request_permissions` (`codex-rs/core/src/tools/handlers/request_permissions.rs:32-36`); description: "Request additional filesystem or network permissions from the user and wait for the client to grant a subset of the requested permission profile. Use environment_id to target a specific attached environment … Granted permissions apply automatically to later shell-like commands in the current turn, or for the rest of the session if the client approves them at session scope." (`codex-rs/core/src/tools/handlers/shell_spec.rs:193-196`).
- Protocol: `RequestPermissionsArgs{environment_id?, reason?, permissions: RequestPermissionProfile{network?, file_system?}}`; response `{permissions, scope: PermissionGrantScope::{Turn, Session}, strict_auto_review}` — "Review subsequent commands in this turn unless a permission hook resolves the request" (`codex-rs/protocol/src/request_permissions.rs:12-75`).
- Per-command variant: `sandbox_permissions: "with_additional_permissions"` + `additional_permissions` ("Sandboxed filesystem or network access for this command"), offered only when exec permission approvals are enabled (`codex-rs/core/src/tools/handlers/shell_spec.rs:235-276`).
- Execution: `SandboxOverride::EscalatedSandboxWithRestrictions` — sandbox stays, profile widened (`codex-rs/core/src/tools/orchestrator.rs:236-262`).
- Approval routing: Granular flag `request_permissions` (false = auto-reject) (`codex-rs/protocol/src/protocol.rs:1043-1067`); guardian request kind `RequestPermissions` (`codex-rs/core/src/guardian/approval_request.rs:18-200`).
- Prompt: `on_request_rule_request_permission.md` tells the model to prefer `with_additional_permissions` (network.enabled / file_system.read / file_system.write) over full escalation; `REQUEST_PERMISSIONS_TOOL` section when the tool is present (`codex-rs/prompts/src/permissions_instructions.rs:29-31`) ([[codex--permission-state-prompt]]).

## Evolution
- 2026-03-08 `e6b93841c5` "Add request permissions tool (#13092)".
- 2026-08-18 `d68b85a097` fresh approval required beneath denied permission paths.

## Versus pi
- No equivalent (no sandbox). Closest UX is pi's generic `ui.confirm` in extensions ([[pi--tool-call-gate]]).
