---
type: tradeoff
concepts: [extension-event-hooks, runtime-plugin-loading, plugin-tools, replaceable-builtin-extension, harness-package-distribution, client-supplied-dynamic-tools, mcp-integration, skill-progressive-disclosure]
---
**Axis** — How third parties extend the harness: runtime-loaded code with full in-process access vs compile-time first-party extensions plus declarative third-party bundles (skills, MCP servers, hooks as processes).

| aspect | pi | codex |
|---|---|---|
| first-party features | built-in extensions on the **public** API, replaceable by name (`builtin:<name>`) ([[pi--replaceable-builtin-extension]]) | compiled `codex-rs/ext/*` crates on an internal contributor API, toggled by feature flags ([[codex--replaceable-builtin-extension]], [[feature-flag-stages]]) |
| third-party code | TS plugins loaded at runtime (jiti, virtual modules, `/reload`) ([[pi--runtime-plugin-loading]]) | none — plugin manifest has no code (`codex-rs/plugin/src/manifest.rs:8-58`) → [[no-executable-plugins]] |
| hooks | 41 typed in-process `pi.on` events, chained transforms, veto ([[pi--extension-event-hooks]]) | external-process command/MCP-tool hooks, Claude-Code-compatible protocol, 600 s timeout (`codex-rs/hooks/src/engine/discovery.rs:764`) ([[codex--extension-event-hooks]]) |
| tools | `pi.registerTool` in-process ([[plugin-tools]]) | MCP servers in bundles ([[mcp-integration]]) + host-declared dynamic tools ([[client-supplied-dynamic-tools]]) |
| UI | `ctx.ui` dialogs/widgets/renderers ([[extension-ui-primitives]]) | declarative `interface` metadata only; interaction via server→client requests |
| packaging | npm/git/URL/path packages with code, pinning, filters ([[pi--harness-package-distribution]]) | marketplaces of declarative bundles; accepts `.claude-plugin/plugin.json` and Agent Plugins 1.0 ([[codex--harness-package-distribution]]); importer for Claude Code / Cursor configs ([[external-agent-import]]) |
| trust/admin | project trust gate; third-party installs run lifecycle scripts | trust gate for project config/hooks/exec policies; managed requirements (plugin/marketplace/MCP allow rules, managed-hooks-only) ([[codex--layered-settings]]) |
| failure classes | stale ctx, leaked registrations, duplicate host modules, module-cache leaks ([[Platform Failures]]) | hook I/O deadlock ([[plugin-hook-wall-clock-timeout]]), marketplace identity spoofing (`0acf302db5`) |

**When each wins**
- **Runtime code plugins (pi)**: maximal flexibility and a minimal core — anything (sub-agents, plan mode, permission gates) can ship as a plugin; best for a single-user tool whose users write code.
- **Compiled extensions + declarative bundles (codex)**: enterprise/managed deployments and a sandboxed agent — no third-party code inside the harness process, admin allow-lists apply to every capability, and bundles are portable across harnesses (Claude Code manifests, MCP, SKILL.md). Cost: third parties cannot change loop behavior except through out-of-process hooks.
