---
type: implementation
harness: codex
concept: layered-settings
commit: 622e9e3696
files: [codex-rs/config/src/config_layer_source.rs:30, codex-rs/config/src/loader/README.md:1, codex-rs/config/src/loader/mod.rs:1093, codex-rs/core/src/config/mod.rs:271, codex-rs/config/src/config_requirements.rs:1030, codex-rs/cloud-config/src/service.rs:37, codex-rs/cloud-config/src/cache.rs:27, codex-rs/config/src/strict_config.rs:1]
---
[[layered-settings]] in [[codex]].

Two stacks: a provenance-tagged **value** stack (`ConfigLayerStack`) and a separate enterprise **constraint** stack (requirements) that values are checked against.

## Mechanism
- **Value layers**, higher precedence wins (`codex-rs/config/src/config_layer_source.rs:30-52`; narrative `codex-rs/config/src/loader/README.md` "Layering model"):

| layer | precedence |
|---|---|
| PackagedDefaults | −10 |
| Mdm | 0 |
| System `/etc/codex/config.toml` | 10 |
| EnterpriseManaged (cloud bundle) | 15 |
| User `$CODEX_HOME/config.toml` | 20 |
| selected profile `<name>.config.toml` | 21 |
| Project `.codex/config.toml` | 25 |
| SessionFlags (CLI `-c key=value`) | 30 |
| LegacyManagedConfigTomlFromFile | 40 |
| LegacyManagedConfigTomlFromMdm | 50 |

- Stack yields **per-key origins** and **per-layer fingerprints** for optimistic-concurrency writes (README "Public surface"); app-server `config/value/write`, `config/batchWrite` ([[codex--client-server-session-split|client-server-session-split]]); user config hot-reloads on batch writes (`a684a36091` 2026-03-08).
- **Trust gate**: the project layer is still loaded but carries a `disabled_reason` when the project is untrusted — "project-local config, hooks, and exec policies" are gated (`codex-rs/config/src/loader/mod.rs:1093-1105`) → [[project-trust-gate]], [[untrusted-repo-loads-executable-config]].
- **Profiles v2**: `--profile NAME` selects `$CODEX_HOME/NAME.config.toml` (`codex-rs/core/src/config/mod.rs:271`, `:2047`); legacy `[profiles.*]` tables removed `fd72e99384` 2026-05-22.
- **Requirements = constraints, not values** (`codex-rs/config/src/config_requirements.rs:1030-1071`): allow-lists (`allowed_approval_policies`, `allowed_sandbox_modes`, `allowed_permission_profiles`, `allowed_web_search_modes`, `allowed_login_methods`, `allowed_chatgpt_workspaces`); exact pins (`sqlite_home`, `log_dir`, `model_catalog_json`, `check_for_update_on_startup`, `allow_login_shell`); feature pins `feature_requirements` (feature→bool, `codex-rs/config/src/config_requirements.rs:891`) → [[feature-flag-stages]]; `allow_managed_hooks_only` (honored only in `requirements.toml`, `docs/config.md`); MCP server / plugin / marketplace / apps allow rules; exec-policy `rules`; `network`; residency; Windows sandbox; browser/computer-use gates; `allow_remote_control`.
- **Runtime enforcement**: values wrapped in `Constrained<T>` (e.g. delegates `Constrained::allow_only(AskForApproval::Never)`, `codex-rs/core/src/codex_delegate.rs:73`); app-server writes overlapping an exact requirement fail with `configRequirementReadonly` (`ee71c4a90f` 2026-07-21); `configRequirements/read` exposes them.
- **Cloud-delivered managed config** `codex-rs/cloud-config`: fetch timeout 20 s, 5 attempts, refresh every 15 min, 5 s timeout-retry interval, on-disk cache `cloud-config-bundle-cache.json` TTL 1 h, HMAC-signed with an embedded key (`codex-rs/cloud-config/src/service.rs:37-46`, `codex-rs/cloud-config/src/cache.rs:27-33`). Requirements layers composed since `20debf746b` 2026-05-31.
- **Schema + strictness**: JSON schema generated from `ConfigToml` via schemars (`codex-rs/config/src/schema.rs`; generator `codex-rs/config-schema/src/main.rs`; checked-in `codex-rs/core/config.schema.json`); `--strict-config` rejects unknown keys (`codex-rs/config/src/strict_config.rs`).
- **Role layers**: agent roles reuse the stack through a bounded override (roles may only narrow) → [[agent-roles]].
- Docs are pointers: `docs/config.md` links to developers.openai.com → [[no-in-repo-user-docs]].

### Config key inventory — safety / state / MCP / profiles (`ConfigToml`, `codex-rs/config/src/config_toml.rs:166`; docs `docs/config.md` is a 15-line pointer to developers.openai.com)
| area | keys (doc-comment gist, line) |
|---|---|
| sandbox / approvals | `approval_policy`, `approvals_reviewer`, `auto_review`, `sandbox_mode`, `sandbox_workspace_write.{writable_roots, network_access, exclude_tmpdir_env_var, exclude_slash_tmp}` (`codex-rs/config/src/types.rs` `SandboxWorkspaceWrite`); `default_permissions` ("Names starting with `:` refer to built-in profiles; other names are resolved from the `[permissions]` table", `:236-239`); `permissions` named profiles (`:243`); `allow_symlinked_codex_home` (macOS-only, user config only, default false: "trusts symlink targets even if they change between commands or lie outside CODEX_HOME", `:226-231`); `shell_environment_policy`, `allow_login_shell`; `[windows] sandbox / allow_mxc` → [[codex--os-level-sandbox]] |
| MCP | `mcp_servers`; `mcp_enterprise_managed_auth` ("Trusted enterprise IdP shared by EMA-enabled MCP servers and plugins", `:300`); `mcp_oauth_credentials_store` keyring / file / auto (default) (`:303-308`); `mcp_oauth_callback_port` (ephemeral if unset, `:312`); `mcp_oauth_callback_url` (listener still binds 127.0.0.1, `:314-318`); `mcp_optional_startup_grace_ms` default 1000, `0` = wait per-server `startup_timeout_sec` (`:320-324`) → [[mcp-integration]] |
| state / history | `history.{persistence, max_bytes}` (`~/.codex/history.jsonl`; oldest dropped past `max_bytes`, `codex-rs/config/src/types.rs:224-230`) → [[codex--cross-session-prompt-history]]; `sqlite_home` (default `$CODEX_SQLITE_HOME` else `$CODEX_HOME`, `:369-371`); `log_dir` (default `$CODEX_HOME/log`, `:373-376`); `thread_unload_delay_secs` (app-server unloads a thread with no subscribers/activity, default 60, `0` = immediately, `:344-348`); `experimental_thread_store` (`:468`) and removed `experimental_thread_store_endpoint` "kept only so we can fail fast instead of silently falling back to local persistence" (`:462-465`); `ghost_snapshot.*` legacy no-ops (`:515`, `:805-814`) → [[no-checkpoints-undo]] |
| auth | `cli_auth_credentials_store`, `forced_login_method` ("restricts the login mechanism users may use", `:283`), `forced_chatgpt_workspace_id` |
| profiles | `profile` / `profiles` still deserialize (`:359-363`) but top-level `profile = "..."` is rejected at load and all profile-v1 merge points were removed (`fd72e99384`); selection is `--profile NAME` → `NAME.config.toml` |
| plugins / skills / hooks | `plugins`, `marketplaces`, `skills`, `hooks`, `apps`, `tool_suggest` → [[plugin-tools]], [[skill-progressive-disclosure]], [[codex--extension-event-hooks]] |
| telemetry | `otel.*`, `analytics`, `feedback` → [[codex--install-telemetry]]; `suppress_unstable_features_warning` (`:510`) → [[codex--feature-flag-stages]] |

## Constants
| name | value | path:line |
|---|---|---|
| layer precedences | −10/0/10/15/20/21/25/30/40/50 | codex-rs/config/src/config_layer_source.rs:33-50 |
| cloud config fetch | 20 s timeout, 5 attempts, 15 min refresh, 5 s retry interval | codex-rs/cloud-config/src/service.rs:37-46 |
| cloud config cache TTL | 1 h | codex-rs/cloud-config/src/cache.rs:29 |

## Evolution
- 2026-03-08 `a684a36091` user config hot-reload on batch writes.
- 2026-05-22 `fd72e99384` legacy `[profiles.*]` removed.
- 2026-05-31 `20debf746b` composed requirements layers.
- 2026-07-21 `ee71c4a90f` `configRequirementReadonly`.
- 2026-08-04 `1e59dc5bda` auto-trust of undecided projects → replaced same day by explicit prompt `17801b4206` ("Trusting a directory enables project-local config, hooks, and exec policies, which can increase exposure to prompt injection").

## Quirks
- Legacy managed-config layers (40/50) outrank CLI session flags (30) — admins beat users even on the command line.

## Versus pi
- [[pi--layered-settings]]: pi = global → project → CLI deep merge with lockfile + modified-field writes; codex = 10 provenance-tagged layers with fingerprints, plus a separate requirements constraint stack (MDM / system / cloud bundle) that pi has no analogue for.
