---
type: failure
concepts: [minimal-default-toolset, mcp-integration]
harnesses: [pi]
---
**Symptom** — `pi --tools read,codemode` removed every MCP tool, so codemode scripts could not reach any MCP server.

**Root cause** — The built-in tool allowlist was applied to *all* registered tools, conflating "which tools are declared to the model" with "which tools are reachable" (MCP tools default to script-only `codemode` exposure).

**Fix · [[pi]]** — `04b97ef00` 2026-10-05: `--tools` no longer removes MCP tools unless an entry starts with `mcp__`; unnamed MCP tools are never declared directly (only tool_search can load them); `*` patterns for `--tools`/`--exclude-tools` (e.g. `'mcp__radius__*'`); `--no-mcp` disables MCP for one run; `--no-tools` still removes everything (`packages/coding-agent/docs/mcp.md:228`, `docs/cli.md:132`; matcher `packages/coding-agent/src/core/mcp-servers.ts:12-25`).

**Lesson** — Separate declaration selection from reachability; an allowlist for the model's tool list must not silently cut tools reached through other channels.

Related: [[minimal-default-toolset]] · [[mcp-integration]] · [[deferred-tool-loading]] · [[pi--minimal-default-toolset|pi]]
