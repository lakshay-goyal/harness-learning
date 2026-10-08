---
type: implementation
harness: pi
concept: layered-settings
commit: b30a6dd77
files: [packages/coding-agent/src/core/settings-manager.ts:195-277, packages/coding-agent/src/core/settings-manager.ts:327-351, packages/coding-agent/src/core/settings-manager.ts:605-627, packages/coding-agent/src/core/settings-manager.ts:660, packages/coding-agent/src/core/settings-manager.ts:685-689, packages/coding-agent/src/core/settings-manager.ts:728-757, packages/coding-agent/docs/settings.md:3, packages/coding-agent/docs/configuration.md:3-47]
---
[[layered-settings]] in [[pi]].

## Mechanism
- Files: global `~/.pi/agent/settings.json` (dir overridable via `PI_CODING_AGENT_DIR`) + project `.pi/settings.json` (`docs/settings.md:3`).
- Merge (`core/settings-manager.ts:195-277`): project overrides global; nested objects deep-merge; arrays replace — EXCEPT resource lists combine, and `defaultTools` containing only `+name/-name` modifiers appends across layers (mixing plain names and modifiers throws, `:223-233`) → [[minimal-default-toolset]].
- CLI overrides on top via `applyOverrides` (`:660`). Env overrides for selected keys (`PI_TELEMETRY`, `PI_OFFLINE`, `PI_CACHE_RETENTION`, …).
- Trust gating: project settings empty until trusted; writes to project settings throw when untrusted (`:605-627,685-689`). `defaultProjectTrust` (`ask|always|never`) is global-only (`packages/coding-agent/docs/settings.md:34`); `sessionDir` is the only project setting read pre-trust (`docs/configuration.md:3`). Project `.pi` settings/resources/packages/extensions gated; context files (AGENTS.md) are NOT (`docs/configuration.md:3,47`) → [[project-trust-gate]].
- Other global-only keys: `cacheWarming` ("because each refresh costs money", `settings-manager.ts:80-82,182`), `compaction.enabled` (`:937-947`).
- Concurrency: saves take `proper-lockfile` (10 attempts × 20 ms busy-wait) and write only fields modified this session (incl. nested keys) merged onto current on-disk content — avoids clobbering concurrent pi instances (`:327-351,728-757`; `9dfe5bf89` merge-on-write #527, `de2736bad`).
- Agent dir siblings: `settings.json`, `keybindings.json`, `mcp.json`, `models.json`, `auth.json`, `AGENTS.md`/`CLAUDE.md`, `SYSTEM.md` (replace), `APPEND_SYSTEM.md`, `extensions/`, `skills/`, `prompts/`, `themes/` (`docs/configuration.md:11-24`). Project `.pi/SYSTEM.md` beats agent-dir one; not combined (`docs/configuration.md:39`) → [[system-prompt-override]].
- Compaction overrides `compaction.modelOverrides` keyed by exact `provider/modelId`; invalid values throw on read (`settings-manager.ts:950-976`).
- Keybindings file → [[pi--differential-tui-rendering]].
- **Malformed settings files**: a parse error records `globalSettingsLoadError`/`projectSettingsLoadError` and `save()` then **skips writing that scope**, so a broken hand-edited file is never overwritten by in-app changes (`core/settings-manager.ts:412-413,759-790`). Errors are drained into non-fatal `warning` diagnostics (`Invalid settings file <path>: …`), deduplicated across the startup and runtime managers by `type\0message` (`packages/coding-agent/src/core/settings-diagnostics.ts:4-25`; `packages/coding-agent/src/main.ts:670,799,914`); runtime creation returns diagnostics instead of printing or exiting, so the app layer decides (`core/agent-session-services.ts:18-28`).
- Small resolution rules: `externalEditor` → `$VISUAL` → `$EDITOR` → `notepad` (Windows) / `nano` (`settings-manager.ts:1077-1087`); the editor opens a temp `pi-editor-*/prompt.md` via async `spawn` — never `spawnSync`, which on Windows races vim for console input (`modes/interactive/external-editor.ts`). `theme` containing `/` is ignored as a name (`settings-manager.ts:879-882`). `deviceId` is global-only "so a committed project settings file cannot give every clone the same ID" (`:1193-1205`). Full key/default table: [[pi#Coverage index]].

## Constants
| name | value | path:line |
|---|---|---|
| settings lock | 10 × 20 ms | `settings-manager.ts:327-351` |
| `enableInstallTelemetry` | true | `settings-manager.ts:155` |
| `enableAnalytics` | false | `settings-manager.ts:156` |
| `tuiMode` | `"fullscreen"` | `settings-manager.ts:1372` |
| `DEFAULT_TOOL_NAMES` | read, bash, edit, write | `settings-manager.ts:215` |

## Evolution
- 2025-11-13 `c82f9f4f8` settings manager created (with changelog-on-startup).
- 2025-12-22 `62c64a286` project-specific settings + factories.
- 2026-02-17 `de2736bad` improved storage semantics (lockfile, modified-field writes).
- 2026-06-05 `89a92207f` project settings gated by trust.
- 2026-09-29 `4259686d9` `builtin:<name>` entries in `extensions`, project `+/-builtin` overrides user.

## Evidence commits
`c82f9f4f8` `62c64a286` `de2736bad` `89a92207f` `4259686d9`

## Quirks
- Locks everywhere (settings, auth, trust, MCP OAuth, server) — almost all `10 × 20ms` sync or ~30 s stale (constants census cross-cutting note).
- Arrays-replace rule has bespoke exceptions; users must know which keys combine.

## Failures
[[concurrent-settings-writes-clobber]]
