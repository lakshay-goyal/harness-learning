---
type: implementation
harness: codex
concept: permission-state-prompt
commit: 622e9e3696
files: [codex-rs/prompts/src/permissions_instructions.rs:29, codex-rs/prompts/src/permissions_instructions.rs:272, codex-rs/prompts/src/permissions_instructions.rs:382, codex-rs/prompts/templates/permissions/approval_policy/on_request.md:1, codex-rs/prompts/templates/permissions/sandbox_mode/workspace_write.md:1, codex-rs/core/src/context/approved_command_prefix_saved.rs:5, codex-rs/core/src/context/world_state/compact_permissions.rs:12]
---
[[permission-state-prompt]] in [[codex]].

## Mechanism
- Renderer `codex-rs/prompts/src/permissions_instructions.rs`; fragment tag `("<permissions instructions>", "</permissions instructions>")` — note the space inside the tag name (`:272`). Body = sandbox text + writable roots + denied reads + approval text. Developer role ([[message-role-layering]]).
- Sandbox templates (one line each, `{{ network_access }}` = restricted/enabled): `codex-rs/prompts/templates/permissions/sandbox_mode/read_only.md` ("The sandbox only permits reading files."), `workspace_write.md` ("The sandbox permits reading files, and editing files in `cwd` and `writable_roots`. Editing files in other directories requires approval."), `danger_full_access.md` ("No filesystem sandboxing - all commands are permitted.").
- Approval templates (`codex-rs/prompts/templates/permissions/approval_policy/`):
  - `never.md` (120 B): "Approval policy is currently never. Do not provide the `sandbox_permissions` for any reason, commands will be rejected."
  - `unless_trusted.md` (152 B): "The harness will require user approval before running commands unless an explicit exec policy rule allows them."
  - `on_request.md` (3,661 B) "# Escalation Requests": command split into segments at `|`, `&&`, `||`, `;`, `(...)`, `$(...)`, each evaluated independently; redirection / substitution / env assignments / globs "will not be evaluated against rules"; "## How to request escalation" (`sandbox_permissions: "require_escalated"`, short question in `justification`, optional `prefix_rule`); rerun with escalation on "likely sandbox-related network error (for example DNS/host resolution, registry/index access, or dependency download failure)"; "## When to request escalation" (writes to dirs like /var, GUI apps, sandbox failures, "a potentially destructive action such as an `rm` or `git reset` that the user did not explicitly ask for", "don't try and circumvent approvals by using other tools"); "## prefix_rule guidance" + "### Banned prefix_rules" (no `["python3"]`, `["python", "-"]`, "NEVER provide a prefix_rule argument for destructive commands like rm", "NEVER provide a prefix_rule if your command uses a heredoc or herestring") + good examples `["npm","run","dev"]`, `["gh","pr","check"]`, `["cargo","test"]` (`on_request.md:1-60`).
  - `on_request_rule_request_permission.md` (1,725 B): prefer `with_additional_permissions` (network.enabled / file_system.read / file_system.write) over full escalation.
- Inline constants (`codex-rs/prompts/src/permissions_instructions.rs:29-41`): `REQUEST_PERMISSIONS_TOOL` ("# request_permissions Tool …"), `AUTO_REVIEW_SUFFIX` ("`approvals_reviewer` is `auto_review` … If a rejection happens, you should proceed only with a materially safer alternative, or inform the user of the risk and send a final message to ask for approval."), `APPROVED_PREFIXES` ("## Approved command prefixes\nThe following prefix rules have already been approved: "), `GRANULAR_INTRO` ("# Approval Requests\n\nApproval policy is `granular`. Categories set to `false` are automatically rejected instead of prompting…"), `GRANULAR_PROMPTED_CATEGORIES` / `GRANULAR_REJECTED_CATEGORIES`, `MAX_PERMISSION_PATH_BYTES = 32 * 1024`, `OMITTED_PERMISSION_PATHS` ("Additional permission paths/globs are omitted. All restrictions still apply; do not use escalation or additional permissions to bypass omitted read denials.").
- Denied reads section: "## Denied filesystem reads\nThe active permission profile denies reading these paths/globs. Do not request escalation or additional permissions to read them; these denials are policy restrictions." (`codex-rs/prompts/src/permissions_instructions.rs:382-396`).
- Catalog override: model's `model_messages.approvals` / `.permissions` replace the bundled template (`ResolvedMessage::Catalog`, `codex-rs/prompts/src/permissions_instructions.rs:300-306`, `:352-362`).
- Incremental updates: newly approved prefixes / network rules appended as small developer notes ("Approved command prefix saved:", `codex-rs/core/src/context/approved_command_prefix_saved.rs:5`; `codex-rs/core/src/context/network_rule_saved.rs`) rather than re-sending the full block (`codex-rs/core/src/context/world_state/compact_permissions.rs:12`).
- `<environment_context>` (user role) also carries `<filesystem><workspace_roots>…<permission_profile type=…>` and `<network enabled=…><allowed_domains>` (`codex-rs/core/src/context/world_state/environment_render_tests.rs:87-123`).
- Related prompt text elsewhere: Default collaboration mode "Never use the `request_user_input` tool for permission requests or permission-related escalations." (`codex-rs/collaboration-mode-templates/templates/default.md`, `a5c581e247`); per-model prompts' git-safety rules (`codex-rs/core/gpt_5_codex_prompt.md:12-19`).

## Constants
| name | value | path:line |
|---|---|---|
| MAX_PERMISSION_PATH_BYTES | 32 KiB | `codex-rs/prompts/src/permissions_instructions.rs:40` |
| fragment tag | `<permissions instructions>` | `codex-rs/prompts/src/permissions_instructions.rs:272` |

## Evolution
- 2025-08-05 `d31e149cb1` first sandbox section in the static base prompt ("Commands that are blocked by sandbox settings will be automatically sent to the user for approval…"; "When approval is denied or a command fails due to a permission error, do not retry the exact command in a different way."); 2025-08-07 `81b148bda2` dropped the no-retry line.
- 2025-08-12 `90d892f4fd` network names ON/OFF → restricted/enabled: "anecdotally less confusing to the model and requires less reasoning to escalate for approval".
- 2025-11-13 `8dcbd29edd` GPT-5.1: "ALWAYS proceed to use the `with_escalated_permissions` and `justification` parameters … prefer requesting approval via the tool over asking in natural language."
- 2025-12-12 `9429e8b219` "do not message the user before requesting approval for the command."
- 2026-01-12 `87f7226cca` "Assemble sandbox/approval/network prompts dynamically (#8961)" — ~35-line section deleted from `3a6a43ff5c:codex-rs/core/prompt.md` and all five gpt_* prompts; developer message "injected at session start and any time sandbox/approval settings change"; `EnvironmentContext` trimmed to cwd, writable roots, shell; removed fallback "If you are not told about this, assume that you are running with workspace-write, network sandboxing ON, and approval on-failure."
- 2026-01-28 `996e09ca24` `prefix_rule` + "don't try and circumvent approvals by using other tools" (XML good/bad examples); 2026-02-03 `968c029471` Markdown list + "### Banned prefix_rules"; 2026-02-04 `8f17b37d06` heredoc ban; 2026-03-20 `7754dd1b89` "…that would allow arbitrary scripting" + unevaluated shell features.
- 2026-02-27 `6046ca19ba` DNS/registry failures as escalation triggers.
- 2026-06-23 `2cf2a6a844` `on_failure.md` deleted with the mode.
- 2026-08-28 `a5c581e247` no `request_user_input` for permissions.

## Quirks
- XML fences for harness boundaries, Markdown for the instructions inside (XML few-shot tags removed `968c029471`) — see [[xml-prompt-boundaries]].

## Versus pi
- pi's prompt says nothing about permissions because there are none ([[pi--minimal-system-prompt]]; [[no-permission-prompts]]).
