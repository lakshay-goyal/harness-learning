---
type: implementation
harness: codex
concept: minimal-default-toolset
commit: 622e9e3696
files: [codex-rs/core/src/tools/spec_plan.rs:123, codex-rs/core/src/tools/spec_plan.rs:199-292, codex-rs/core/src/tools/spec_plan.rs:392-450, codex-rs/core/src/tools/spec_plan.rs:677-713, codex-rs/core/src/tools/spec_plan.rs:772-861, codex-rs/core/src/tools/spec_plan.rs:1082-1374, codex-rs/core/src/tools/spec_plan.rs:1507, codex-rs/features/src/lib.rs, codex-rs/protocol/src/openai_models.rs:411-510]
---
[[minimal-default-toolset]] in [[codex]] — **contrasting design**: not a fixed tiny default but a per-turn **feature/model/provider-gated tool plan**. What it shares with pi is the small *core* (shell + one edit tool); everything else is computed.

## Mechanism
- `build_tool_router` (`codex-rs/core/src/tools/spec_plan.rs:123`) rebuilds the registry every turn (per step snapshot) → `add_core_tool_sources` (`:1082`) adds shell, MCP-resource, utility and collaboration tools; then hosted tools (`:677-706`), extension tools (`:772-800`, `:1568-1596`), dynamic tools (`append_dynamic_tool_runtimes`, `:1507`), MCP tools (`apply_mcp_tool_exposure_policy`, `:199`).
- **Kill switch**: if tool policy `require_managed_sandbox` is set but the thread or any selected environment's permission profile is not `Managed`, the whole core set is dropped (`:1084-1095`) → [[os-level-sandbox]].
- Gate sources:
  1. `Feature::*` flags (`codex-rs/features/src/lib.rs`, stages UnderDevelopment / Experimental / Stable / Deprecated / Removed → [[feature-flag-stages]]).
  2. Per-model catalog `ModelInfo`: `shell_type`, `apply_patch_tool_type`, `supports_search_tool`, `experimental_supported_tools`, `use_responses_lite`, `web_search_tool_type` (`codex-rs/protocol/src/openai_models.rs:411-510`) → [[model-catalog]].
  3. Provider capabilities (`provider.capabilities().web_search / image_generation`).
  4. Session source (root vs sub-agent).
  5. Environment attached? `advertise_environment_tools = stable_environment_tools || environment_mode.has_environment()` (`:1154-1156`); env-bound tools vanish when exec server is none (`a504d8f0fa`).
  6. Config: `tools.update_plan.enabled` (default false), `tools.experimental_request_user_input.enabled` (default true), tool policy `require_unified_exec`.
- Collisions: `tool_registry.error_on_tool_collisions` makes a duplicate model-visible name a hard turn error `ToolCollision("namespace.name")` before sampling (`:443-450`; `1e489adad0`); any non-search tool claiming `tool_search` is removed and recorded as a collision (`:392-427`) → [[mcp-tool-name-collision]].

### Full tool inventory at `622e9e3696`
| tool | file | gate (default) |
|---|---|---|
| `exec_command` + `write_stdin` | `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:114`, `…/write_stdin.rs:41`; spec `codex-rs/core/src/tools/handlers/shell_spec.rs:96,146` | env && `Feature::ShellTool` (Stable, on) && `shell_type != Disabled`; `Feature::UnifiedExec` (Stable, on) → both, else `ExecCommandHandler::one_shot` without `write_stdin` (`spec_plan.rs:1150-1199`). `tty` param iff `UnifiedExecTty` (on); `shell`/`login` conditional; `environment_id` only with multiple environments → [[shell-execution]] |
| `apply_patch` (freeform, Lark) | `codex-rs/core/src/tools/handlers/apply_patch.rs:286`; spec `apply_patch_spec.rs:9` | env && `apply_patch_tool_type.is_some()` (`:1344-1347`) → [[patch-envelope-edit]] |
| `update_plan` | `codex-rs/core/src/tools/handlers/plan.rs:50` | `config.update_plan_enabled` (**off** since `a9519cbcdd`) (`:1228`) → [[task-list-tool]] |
| `view_image` | `codex-rs/core/src/tools/handlers/view_image.rs:74` | env && `Feature::ViewImage` (Stable, on) (`:1358`) → [[file-read-tool]] |
| `request_user_input` | `codex-rs/core/src/tools/handlers/request_user_input.rs:30` | `experimental_request_user_input_enabled` (on), `DirectModelOnly`, mode-scoped (`:1244-1251`) → [[ask-user-tool]] |
| `request_user_input_async` / `send_message_to_user_async` | — | root agent only; catalog `experimental_supported_tools` or `Feature::SendMessageToUserAsync` (off) (`:1253-1289`) |
| `request_permissions` | spec `shell_spec.rs:180-196` | env && `Feature::RequestPermissionsTool` (off) (`:1291`) → [[model-requested-permissions]] |
| `new_context` (DirectModelOnly) + `get_context_remaining` | `new_context_window_spec.rs:6`, `get_context_remaining_spec.rs:8` | `Feature::TokenBudget` (off) (`:1295-1298`) → [[model-requested-context-reset]] |
| `clock.curr_time` / `clock.sleep` | `current_time.rs:24-25`, `sleep.rs:26-27` | `CurrentTimeReminder` (off) or catalog `clock`; sleep also `Feature::SleepTool` + `SleepToolMode` (`:1300-1326`) → [[wall-clock-tools]] |
| `wait_for_environment` | — | `Feature::DeferredExecutor` (off) (`:1232`) |
| `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource` | `codex-rs/core/src/tools/handlers/mcp_resource/*.rs` | any MCP server configured or tool mode CodeModeOnly (`:1202-1217`) → [[mcp-integration]] |
| `tool_search` (hosted type, client-executed) | `codex-rs/tools/src/tool_discovery.rs:6` | `model_info.supports_search_tool` && ≥1 deferred searchable tool (`:392-427`) → [[deferred-tool-loading]] |
| `list_available_plugins_to_install` + `request_plugin_install` | `codex-rs/core/templates/search_tool/request_plugin_install_description.md` | `Feature::ToolSuggest` (on) && Apps && Plugins && candidates non-empty (`:708-713,1328-1342`) |
| `test_sync_tool` | — | test-only; catalog advertises it (`:1349-1356`) |
| multi-agent v1 (`spawn_agent`, `send_input`, `resume_agent`, `wait_agent`, `close_agent`) / v2 (`spawn_agent`, `send_message`, `followup_task`, `interrupt_agent`, `wait_agent`, `list_agents`) | `add_collaboration_tools` (`:1374`) | `Feature::Collab` (Stable, on); v2 `MultiAgentV2` (Stable, off) → [[task-owned-subagent]], [[in-process-subagent-threads]] |
| code mode `exec` (freeform JS) + `wait` | `codex-rs/code-mode-protocol/src/lib.rs:54-55` | tool mode CodeMode / CodeModeOnly (`Feature::CodeMode*` off) (`:838-861`) → [[code-mode]] |
| hosted `web_search` | `codex-rs/core/src/tools/hosted_spec.rs:14` | provider capability, not Responses Lite, no `web.run` (`:677-706`); mode default Cached → [[web-tools]] |
| `web.run` (extension) | `codex-rs/ext/web-search/src/tool.rs:41-44` | `Feature::StandaloneWebSearch` (off) or Responses Lite (`:1104-1112`) |
| `image_gen.imagegen` (extension) | `codex-rs/ext/image-generation/src/tool.rs:58-63` | `Feature::ImageGeneration` (Stable, on) + provider capability + image-input model (`:772-800`) |
| goal ext `get_goal` / `create_goal` / `update_goal` | `codex-rs/ext/goal/src/spec.rs:9-11` | goals feature → [[persistent-goal-continuation]] |
| `memories.*`, `skills.*`, `history.*` / `notes.*`, agent message board | `codex-rs/ext/*` | extension-registered (`daa48072f4` history/notes) → [[cross-session-memory]], [[skill-progressive-disclosure]], [[agent-message-board]] |
| dynamic tools | `codex-rs/protocol/src/dynamic_tools.rs:1-40` | client-supplied at thread start → [[client-supplied-dynamic-tools]] |
| MCP tools `mcp__<server>__<tool>` | `codex-rs/core/src/tools/handlers/mcp.rs` | per server; Deferred when model supports `tool_search` → [[mcp-integration]] |

- **Typical default (OpenAI model, local env, no MCP)**: `exec_command`, `write_stdin`, `apply_patch`, `view_image`, `request_user_input` (only usable in allowed modes), multi-agent tools when `collab_tools_enabled` (exact default conditions not traced), hosted `web_search` (cached), `image_gen.imagegen` where supported, plugin-install tools when candidates exist. `update_plan` off.
- **No read / grep / ls / write / edit tools** ([[no-file-read-write-tools]]; removed shells: [[removed-legacy-shell-tools]]) — reading and searching via `exec_command`, writing only via `apply_patch` (no `read_file` handler under `codex-rs/core/src/tools/handlers/`). Experimental `list_dir` / `grep_files` / `read_file` existed 2025-10 → removed ([[search-tools]], [[file-read-tool]], [[minimal-vs-rich-toolset]]).
- Code-mode-only sessions hide nested tools from direct exposure; `ToolExposure` levels Direct / Deferred / DeferredModelOnly / DirectModelOnly / CodeModeOnly / Hidden (`codex-rs/tools/src/tool_executor.rs:51-78`).

## Evolution
- 2025-04 TS era: one-shot `shell` tool; 2025-07-29 `8828f6f082` plan tool; 2025-08-23 `363636f5eb` web search; 2025-09-10 `c09ed74a16` unified exec.
- 2025-10-03 `33d3ecbccc` tool registry refactor; `e0b38bd7a2` `beta_supported_tools` (per-model tool lists, later `experimental_supported_tools`).
- 2025-10-05 `f3b4a26f32` "drop read-file for gpt-5-codex"; 2025-10-07..09 `226215f36d` list_dir, `f52320be86` grep_files, `0026b12615` read_file indentation mode (experimental).
- 2026-03-25 `14c35a16a8` / `178c3b15b4` read_file / grep_files handlers removed; 2026-05-05 `70807730f5` list_dir removed ("nothing in the current model catalog advertises it via `experimental_supported_tools`").
- 2026-04-06 `a504d8f0fa` env-bound tools disabled when exec server is none.
- 2026-05-08 `e783341b70` function-style apply_patch deleted.
- 2026-05-13 `83decfa300` "Remove unused legacy shell tools (#22246)" — raw `shell`, Responses built-in `local_shell`, `container.exec` ("Recent session history showed no active use").
- 2026-08-20 `8a40095ea3` / `bce5f2fcfc` "Standardize shell execution on unified exec" — `shell_command` removed (116 files, 4,704 deletions); legacy model metadata `default`/`local`/`shell_command` mapped to unified exec.
- 2026-08-28 `b836aecd4d` one-shot exec preserved when unified exec disabled by managed config.
- 2026-08-31 `a9519cbcdd` update_plan opt-in.

## Quirks
- The tool set can change mid-session (features, MCP startup, environments) — handled by world-state diffs / incremental tool catalogs rather than rewriting the prefix ([[transcript-carried-system-prompt]], [[world-state-diff-injection]], [[tool-loadout-stale-within-run]]).
- Removals are evidence-driven ("no active use" in session history; "nothing in the catalog advertises it").

## Versus pi
pi: fixed 4-tool default (read/bash/edit/write), 8 built-ins, `--tools` allowlist / `+x/-x` modifiers ([[pi--minimal-default-toolset]]). codex: 2 always-on core tools (exec + patch) plus ~20 gated tools; selection is by feature flags, model catalog and provider capabilities, not a user allowlist. Axis: [[minimal-vs-rich-toolset]].
