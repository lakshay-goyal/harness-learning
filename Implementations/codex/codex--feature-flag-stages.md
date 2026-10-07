---
type: implementation
harness: codex
concept: feature-flag-stages
commit: 622e9e3696
files: [codex-rs/features/src/lib.rs:47, codex-rs/features/src/lib.rs:982, codex-rs/features/src/lib.rs:425, codex-rs/features/src/lib.rs:1422, codex-rs/features/src/feature_configs.rs:289, codex-rs/config/src/config_requirements.rs:891]
---
[[feature-flag-stages]] in [[codex]].

## Mechanism
- `Stage` (`codex-rs/features/src/lib.rs:47-62`): UnderDevelopment "still under development, not ready for external use"; Experimental{name, menu_description, announcement} "made available to users through the `/experimental` menu"; Stable "The feature flag is kept for ad-hoc enabling/disabling"; Deprecated "should not be used anymore"; Removed "useless but kept for backward compatibility reason".
- Registry `pub const FEATURES: &[FeatureSpec]` (`codex-rs/features/src/lib.rs:982`). Surfaces: `[features]` config table, CLI `codex features`, TUI `/experimental`, app-server `experimentalFeature/*`.
- **Census at `622e9e3696`** (parsed from `codex-rs/features/src/lib.rs`): 168 flags = 67 UnderDevelopment, 4 Experimental, 52 Stable (most default-on), 4 Deprecated, 41 Removed. Stage can be platform-conditional: `prevent_idle_sleep` is Experimental on macOS/Linux/Windows, UnderDevelopment elsewhere (`codex-rs/features/src/lib.rs:1953-1965`).
- **Removed keys = abandoned designs**: `undo` ("Removed compatibility flag retained as a no-op so old configs can still parse `undo`", `codex-rs/features/src/lib.rs:425-428`, Stage at `:1006-1009`), `js_repl` (`:429-430`), `search_tool`/`tool_search`, `codex_git_commit`, `apply_patch_freeform`, `remote_models`, `multi_agent_mode`, `enable_fanout` (`:1466`), `send_async_message`, `collaboration_modes`, `personality`, `steer`, `responses_websockets(_v2)`, `tui_app_server` → [[Absences]].
- **Multi-agent flags**: `multi_agent` (Feature::Collab) Stable default **on** (`codex-rs/features/src/lib.rs:1422-1427`); `multi_agent_v2` Stable default **off** (`:1428-1433`); `agent_message_board` UnderDevelopment (`:1454`); `DeferMailboxPreemption` under development, default off (`:1446-1451`); `goals` Stable default on (`:1832`).
- **Feature configs** beside flags (`codex-rs/features/src/feature_configs.rs`, e.g. `MultiAgentV2ConfigToml` at `:289`).
- **Pins**: managed `feature_requirements` map feature→bool (`codex-rs/config/src/config_requirements.rs:891`) → [[codex--layered-settings|layered-settings]]; agent roles may only turn a small set off → [[agent-profiles]].

## Registry inventory (flags not discussed in other notes)
Stage UD / Exp / Stable / Dep / Rem; default on/off; `:N` = key line in `codex-rs/features/src/lib.rs`. Descriptions are the enum doc comments.
| family | flags |
|---|---|
| shell / exec ([[shell-execution]]) | `shell_zsh_fork` UD off `:1050` (route shell tool through the zsh exec bridge); `unified_exec_zsh_fork` Rem on `:1056` (composition gate only, must not turn on either parent flag); `write_stdin_approval` Stable on `:1300` (approval before writing stdin to *escalated* unified-exec terminals); `exec_permission_approvals` UD off `:1294` (exec tools request extra permissions while staying sandboxed); `cwd_relative_turn_diffs` UD off `:1096` |
| code mode ([[code-mode]]) | `code_mode_tool_search` UD `:1132` (ranked tool discovery inside JS); `code_mode_prewarm` UD `:1150` (connect host at session start); `code_mode_tool_description_first` UD `:1162`; `code_mode_only_strict_3p_tools` UD `:1180` (keep MCP/app/dynamic tools deferred in code-mode-only); `code_mode_buffered_exec` Rem `:1138`; `js_repl_tools_only` Rem `:1186` |
| tools / MCP ([[mcp-integration]], [[deferred-tool-loading]]) | `enable_mcp_apps` UD `:1484`; `mcp_2026_07_28` / `codex_apps_mcp_2026_07_28` UD `:1490`/`:1496` (MCP protocol 2026-07-28); `mcp_oauth_refresh_coordination` UD `:1502`; `use_xaa` UD `:1508` (enterprise refresh-token authorization for MCP resources); `apps_mcp_path_override` Rem `:1514`; `deferred_tool_world_state` UD `:1532` (describe deferred namespaces in world state); `unavailable_dummy_tools` Rem `:1550`; `auth_elicitation` Stable on `:1886` (connector auth failures via MCP URL elicitation); `default_mode_request_user_input` UD `:1748`; `web_search_request` / `web_search_cached` Dep `:1198`/`:1204`; `content_item_kinds` Stable on `:1108` / `executed_tool_call_metadata` UD `:1114` (internal Responses metadata) |
| model / transport | `api_key_model_discovery` Stable on `:1366`; `enable_request_compression` Stable on `:1384` (zstd request bodies to codex-backend); `respect_system_proxy` UD `:1412`; `system_proxy_fallback` Stable on `:1418` (retry bootstrap requests via system proxy); `responses_websockets_v2` Rem `:1982`; `fast_mode` / `ultrafast_mode` Stable on `:1910`/`:1916`; `concurrent_reasoning_summaries` UD `:1712`; `use_agent_identity` UD `:2006`; `cli_daybreak` UD `:1372`; `bedrock_setup_wizard` UD `:1892` |
| images | `resize_all_images` Rem (on) `:1700`; `item_ids` Rem (on) `:1706`; `image_detail_original` Rem `:1940`; `omit_app_server_notification_media` UD `:1682` |
| context / prompting / skills | `context_management` UD `:1844`; `terminal_visualization_instructions` UD `:1766`; `skill_search` Stable on `:1724` (shadow-mode skill search, metrics only); `skill_env_var_dependency_prompt` Rem `:1730`; `mentions_v2` Stable on `:1736` (unified TUI mention popup) |
| memory / state ([[session-tree]], [[session-migration]]) | `chronicle` UD `:1270` (sidecar for passive screen-context memories); `external_agent_memory_import` UD `:1246`; `local_thread_store_compression` UD `:1252`; `local_thread_store_shared_compression` Rem `:1258`; `background_paginated_rollout_migration` UD `:1264`; `transcript_v2` Dep `:1001` (use `tui.fullscreen_transcript`) |
| safety ([[os-level-sandbox]], [[secret-handling]]) | `secret_auth_storage` Stable, default `cfg!(windows)` `:1032` (CLI auth in encrypted secrets backend); `use_linux_sandbox_bwrap` Rem `:1318` (bwrap now default); `windows_sandbox_service` UD `:1348` (provision via installed service) |
| Guardian ([[llm-approval-reviewer]]) | `guardianv2` UD `:1814`; `guardianv2_decisions_comparison` UD `:1820` (shadow "Decisions" run, no effect on approvals); `guardian_reuse_parent_compaction` Stable on `:1784`; `guardian_root_handoff_context` UD `:1790`; `guardian_enhanced_node_repl_transcripts` / `guardian_node_repl_transcript_images` UD `:1796`/`:1802`; `guardian_conversation_history_tools` UD `:1808`; `guardianv2.thread_context` Rem `:1778`; `guardian_ext` Rem `:1826` |
| multi-agent ([[subagent-config-inheritance]]) | `model_catalog_in_context` UD `:1436` (spawn model choices in append-only context, not tool descriptions — cache-friendly); `multi_agent_v2_dynamic_tools` UD `:1442` (fresh V2 children inherit client dynamic tools) |
| plugins ([[plugin-tools]]) | `recommended_plugins` Stable **off** `:1562`; `remote_plugin` Stable on `:1658`; `plugin_sharing` Stable on `:1664`; `executor_capability_discovery` UD `:1574` (one exec-server RPC for plugin+skill manifests); `skip_host_skill_discovery` UD `:1580`; `plugin_hooks` Rem `:1586`; `external_migration` Rem `:1670` |
| desktop requirements-only gates | Stable, default on, doc "Requirements-only gate: this should be set from requirements, not user config" (`:281`): `in_app_browser` `:1592`, `browser_annotation_api` `:1598`, `in_app_chat` `:1604`, `in_app_dictation` `:1610`, `in_app_voice` `:1616`, `in_app_local_automation` `:1622`, `in_app_updates` `:1628`, `browser_use` `:1634`, `browser_use_full_cdp_access` `:1640`, `browser_use_external` `:1646`, `computer_use` `:1652`. No Rust consumer outside `codex-rs/features` + config tests → read by desktop clients (inferred) |
| platform / TUI misc | `analytics_plan_history` Exp off `:985` (/analytics allowance history); `daemon_auto_start` Stable on `:995` (shared local daemon for interactive launches → [[codex--client-server-session-split]]); `runtime_metrics` UD `:1228`; `workspace_dependencies` Stable on `:2012`; `prevent_idle_sleep` Exp/UD off `:1952`; `terminal_resize_reflow` Rem (on) `:1192`; `workspace_owner_usage_nudge` Rem `:1970` |

## Constants
| name | value | path:line |
|---|---|---|
| flags by stage (UD/Exp/Stable/Dep/Removed) | 67 / 4 / 52 / 4 / 41 (168 total) | codex-rs/features/src/lib.rs (census) |

## Evolution
- `multi_agent`: enable/revert pair 2026-02-09 `284c03ceab` / `c2bfd1e473`; stabilized `36dfb84427` 2026-03-13.
- `multi_agent_v2` Stable (default off) `b00c9b2e16` 2026-07-20.
- 2026-03-19 → 04-02 features extracted into own crate.

## Versus pi
- pi has no feature registry; behavior toggles are settings keys and experimental code paths are gated by env (`PI_EXPERIMENTAL=1`, [[pi--client-server-session-split]]).
