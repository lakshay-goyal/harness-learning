---
type: implementation
harness: codex
concept: external-agent-import
commit: 622e9e3696
files: [codex-rs/external-agent-migration/src/lib.rs:1, codex-rs/external-agent-migration/src/source_cla.rs:1, codex-rs/external-agent-migration/src/source_cur.rs:19, codex-rs/app-server-protocol/src/protocol/v2/config.rs:774]
---
[[external-agent-import]] in [[codex]].

## Mechanism
- Crate "Migration helpers for importing external-agent configuration into Codex" (`codex-rs/external-agent-migration/src/lib.rs:1`); modules: detect, source/source_cla/source_cur, mcp, subagents, hooks_cla/hooks_cur/hooks_common, plugins, memory/memory_import, sessions (append, export, ledger, records_cla/records_cur, title), rewrite, scope, config_values, reporting.
- **Item types** (`ExternalAgentConfigMigrationItemType`, `codex-rs/app-server-protocol/src/protocol/v2/config.rs:774-806`): AGENTS_MD, CONFIG, SKILLS, PLUGINS, MCP_SERVER_CONFIG, SUBAGENTS, HOOKS, COMMANDS, MEMORY, SESSIONS.
- **Sources**: two named external harnesses — Claude Code (`source_cla.rs`) and Cursor (`source_cur.rs`) — with rewrite profiles for hooks/commands; knows Claude Code marketplace names (`claude-plugins-official` → `anthropics/claude-plugins-official`, `claude-code-plugins` → `anthropics/claude-code`) and `.cursor-plugin/marketplace.json` (`codex-rs/external-agent-migration/src/source_cur.rs:19`).
- **Surfaces**: app-server `externalAgentConfig/detect` / `externalAgentConfig/import`, TUI `/import`.
- Complements load-time compatibility: `.claude-plugin/plugin.json` manifests ([[codex--harness-package-distribution|harness-package-distribution]]) and the `ClaudeHooksEngine` hook protocol ([[codex--extension-event-hooks|extension-event-hooks]]).

## Evolution
- 2026-04-28 `cb8b1bbcd6` "Support detect and import MCP, Subagents, hooks, commands from external"; path-rewrite fix next day `d92c909ee4`.

## Versus pi
- pi has no importer; it reads other harnesses' instruction files only through its own context-file rules ([[context-file-hierarchy]]).
