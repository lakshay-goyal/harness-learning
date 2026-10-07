---
type: concept
stage: architecture
tier: candidate
aliases: [codex-external-agent-migration, externalAgentConfig/detect, externalAgentConfig/import, /import, ExternalAgentConfigMigrationItemType]
harnesses: [codex]
---
Detect another coding-agent harness's configuration on the machine (instructions files, config, skills, plugins/marketplaces, MCP servers, sub-agents, hooks, commands, memory, sessions) and translate it into the importing harness's equivalents.

## Why
- Switching cost: users have invested in another harness's MCP servers, hooks, commands and instruction files; migration by hand is error-prone.
- Formats are near-equivalent but not identical (paths, hook events, marketplace names) — needs rewrite profiles per source.

## Design space
- **Compatibility at load time** (read foreign manifests directly — codex `.claude-plugin/plugin.json`, Claude-compatible hooks engine) vs **one-shot import/translation** (codex external-agent-migration) vs none (pi).
- **Item granularity**: per category with user selection (codex item types AGENTS_MD, CONFIG, SKILLS, PLUGINS, MCP_SERVER_CONFIG, SUBAGENTS, HOOKS, COMMANDS, MEMORY, SESSIONS).
- **Rewrite**: verbatim copy vs per-source path/event rewrite profiles (codex).
- **Sessions**: ignore vs import transcripts with a ledger to avoid duplicates (codex `sessions/ledger.rs`).

## Implementations
- [[codex--external-agent-import|codex]] — `codex-rs/external-agent-migration` with two sources (Claude Code, Cursor), app-server `externalAgentConfig/detect|import`, TUI `/import`.

## Failures
- (none mined)

## Related
[[harness-package-distribution]] · [[layered-settings]] · [[turn-lifecycle-hooks]] · [[mcp-integration]] · [[context-file-hierarchy]] · [[extensibility-model]]
