---
type: implementation
harness: opencode
concept: replaceable-builtin-extension
commit: ecc4916b5a
files: [packages/opencode/src/plugin/index.ts:67-86, packages/opencode/src/plugin/index.ts:170-182, packages/core/src/plugin/internal.ts:108-123, packages/core/src/plugin/agent.ts:97-160, specs/v2/instructions.md:7]
---
[[replaceable-builtin-extension]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Provider auth as built-in plugins**: Codex, Copilot, Modal, GitLab, Poe, Cloudflare Workers, Cloudflare AI Gateway, Azure, DigitalOcean, Snowflake Cortex, xAI auth plugins and a Cerebras plugin are written against the public `@opencode-ai/plugin` API and loaded before user plugins (`packages/opencode/src/plugin/index.ts:67-86`).
- **Disable**: all at once with `OPENCODE_DISABLE_DEFAULT_PLUGINS` (`packages/opencode/src/plugin/index.ts:170`); `OPENCODE_PURE` disables external plugins instead (`packages/opencode/src/plugin/index.ts:181-182`). No per-plugin toggle or same-name replacement rule found.
- Core features (tools, agents, MCP, LSP, share) stay in core.

### v2 runtime
- **Plugin-first core**: "Move behavior out of large application services and into plugins" (`specs/v2/instructions.md:7`). `PluginInternal` boots built-in plugins in order: config references, agents, commands, skills, models.dev catalog, config-driven agents/commands/skills, provider plugins, external plugins, config providers, variants (`packages/core/src/plugin/internal.ts:108-123`).
- Built-in agents (`build`, `plan`, `general`, …) are defined by the `agent` plugin through `ctx.agent.transform` (`packages/core/src/plugin/agent.ts:97-160`), so later plugins (including config) transform the same drafts ([[agent-profiles]]).

## Constants
| name | value | path:line |
|---|---|---|
| legacy internal plugins | 12 | `packages/opencode/src/plugin/index.ts:70-84` |

## Evolution
- 2026-06-03 `76ee87ead8` v2 runtime; built-ins moved behind the plugin API there.

## Quirks / drift
- Legacy disable switch is all-or-nothing, so turning off one provider auth plugin disables Copilot/Codex auth too.

pi contrast: named `builtin:<name>` resources (MCP, codemode, tool-search, llama.cpp) that step aside when a third-party extension registers the same tool ([[pi--replaceable-builtin-extension|pi]]).
