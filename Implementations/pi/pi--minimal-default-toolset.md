---
type: implementation
harness: pi
concept: minimal-default-toolset
commit: b30a6dd77
files: [packages/coding-agent/src/core/settings-manager.ts:215, packages/coding-agent/src/core/tools/index.ts:96, packages/coding-agent/src/core/system-prompt.ts:64, packages/coding-agent/src/core/agent-session.ts:3651, packages/coding-agent/src/core/sdk.ts:64, packages/coding-agent/src/main.ts:540, packages/coding-agent/src/extensions/index.ts:7, packages/coding-agent/src/core/mcp-servers.ts:12, packages/durable/src/tools/index.ts:21]
---
[[minimal-default-toolset]] in [[pi]].

## Mechanism
- Default active set = 4 tools: `DEFAULT_TOOL_NAMES = ["read","bash","edit","write"]` (`packages/coding-agent/src/core/settings-manager.ts:215`). Same list hard-coded twice more as fallbacks: `selectedTools ?? ["read","bash","edit","write"]` in the prompt builder (`packages/coding-agent/src/core/system-prompt.ts:64`) and `defaultActiveToolNames` in session (`packages/coding-agent/src/core/agent-session.ts:3651`) — three copies, no shared constant.
- Built-in tool universe = 8: `ToolName = "read"|"bash"|"powershell"|"edit"|"write"|"grep"|"find"|"ls"`, `allToolNames` (`packages/coding-agent/src/core/tools/index.ts:95-105`). Factory bundles: `createCodingTools` = read,bash,edit,write (`index.ts:195-202`); `createReadOnlyTools` = read,grep,find,ls (`index.ts:204-211`).
- Built-in **extensions** add tools that are registered but not active: `codemode` (`defaultActive:false`, `packages/coding-agent/src/extensions/codemode/index.ts:43`), `tool_search` (`packages/coding-agent/src/extensions/tool-search/index.ts:14`), MCP tools (default exposure `codemode` = never declared), MCP resource tools (only when some server has resources). `llama.cpp` registers no tools (`packages/coding-agent/src/extensions/llama/index.ts:44,183`). codemode/tool-search/mcp are `replaceable: true` (`packages/coding-agent/src/extensions/index.ts:7-14`) → [[replaceable-builtin-extension]].
- Auto-activation exceptions: codemode auto-activates when a configured MCP server uses `codemode` exposure (unless `autoEnableCodemode === false`); `tool_search` auto-activates for `deferred` exposure (`ensureDiscoveryActive`, `packages/coding-agent/src/extensions/mcp/index.ts:481-519`). See [[pi--mcp-integration|pi MCP]].
- **Inventory** (schema gist; details in per-tool notes):

| tool | default | params | detail note |
|---|---|---|---|
| `read` | on | `path`, `offset?`, `limit?` | [[pi--file-read-tool]] |
| `bash` | on | `command`, `timeout?` (s) | [[pi--shell-execution]] |
| `edit` | on | `path`, `edits[{oldText,newText}]` | [[pi--search-replace-edit]] |
| `write` | on | `path`, `content` | [[pi--search-replace-edit]] |
| `grep` | off | `pattern`, `path?`, `glob?`, `ignoreCase?`, `literal?`, `context?`, `limit?`=100 | [[pi--search-tools]] |
| `find` | off | `pattern`, `path?`, `limit?`=1000 | [[pi--search-tools]] |
| `ls` | off | `path?`, `limit?`=500 | [[pi--search-tools]] |
| `powershell` | off, Windows only | as bash | [[pi--shell-execution]] |
| `codemode` | off (auto w/ MCP codemode exposure) | `code` (raw JS, Lark grammar) | [[pi--code-mode]] |
| `tool_search` | off (auto w/ deferred exposure) | `query`, `limit?`=8 | [[pi--deferred-tool-loading]] |
| `mcp__<server>__<tool>` | per exposure (default not declared) | server schema | [[pi--mcp-integration]] |
| `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource` | only if a server has resources | — | [[pi--mcp-integration]] |

- **Selection grammar** (`defaultTools` setting / `--tools` / SDK `tools`): either a plain allowlist (names or `*` patterns) **or** only `+name`/`-name` modifiers with exact names; mixing → error "tool names cannot be mixed with +name or -name entries"; pattern in a modifier → error (`getToolListError`, `settings-manager.ts:223-233`). `applyToolModifiers` applies `+`/`-` in order (`:239-249`). Layer merge: plain list replaces inherited, modifier-only list is **appended** to inherited (`mergeDefaultTools` `:253-259`); `resolveDefaultTools` = plain names (or `DEFAULT_TOOL_NAMES` if none) then modifiers (`:264-270`). See [[layered-settings]].
- Pattern matcher: `*` → `.*`, anchored regex (`packages/coding-agent/src/core/mcp-servers.ts:12-25`).
- `--exclude-tools` denylist with patterns (`packages/coding-agent/src/core/sdk.ts` `excludeTools`).
- `--no-tools` → `noTools:"all"` (nothing enabled); `noTools:"builtin"` disables the 4 defaults but keeps extension/custom tools (`packages/coding-agent/src/core/sdk.ts:64-71`, `packages/coding-agent/src/main.ts:540-543`). Empty tool list renders `(none)` in prompt `<tools>` (`8f5523ed5`).
- `--tools` does NOT remove MCP tools unless an entry starts with `mcp__` (`04b97ef00`; `packages/coding-agent/docs/mcp.md:228`, `docs/cli.md:132`). Opt-in examples: `"defaultTools": ["+tool_search"]` (`docs/cli.md:182`), `["+codemode"]` (`docs/mcp.md:204,230`).
- Active set == declared set (`packages/coding-agent/src/core/extensions/types.ts:504-516`); exposure tiers in [[deferred-tool-loading]].
- Rationale stated: grep/find/ls are "read-only exploration tools … safe code exploration without modification risk" (`186169a82`); embeddings/RAG/repo-map absent by omission — agent expected to explore with bash/rg (`ca/docs/settings.md:40,44`) → [[no-codebase-index]]. Historical "No MCP" stance: "Token efficient: A 225-token README beats a 13,000-token MCP server description"; "Composable: Chain tools with bash pipes" (removed lines in `60e4fcf01`).
- Prompt coupling: when bash/powershell present and none of grep/find/ls → rule "Use bash for file operations like ls, rg, find" (`packages/coding-agent/src/core/system-prompt.ts:108-116`); see [[dynamic-tool-guidelines]].

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_TOOL_NAMES` | read, bash, edit, write | `packages/coding-agent/src/core/settings-manager.ts:215` |
| `allToolNames` | 8 built-ins | `packages/coding-agent/src/core/tools/index.ts:96-105` |
| prompt fallback selectedTools | read, bash, edit, write | `packages/coding-agent/src/core/system-prompt.ts:64` |
| session fallback active tools | read, bash, edit, write | `packages/coding-agent/src/core/agent-session.ts:3651` |
| durable `CodingTools` | read, write, edit, bash (+`createPowerShellTool()`) | `packages/durable/src/tools/index.ts:21-25` |

## Evolution
- 2025-10-17 `ffc9be886` — v0: 4 tools read/bash/edit/write; prompt "Always use bash tool for file operations like ls, grep, find".
- 2025-11-12 `60e4fcf01` — README: "pi does not support MCP… relies on the four built-in tools above and assumes the agent can invoke pre-existing CLI tools or write them on the fly".
- 2025-11-29 `186169a82` — grep/find/ls added opt-in + `--tools` flag; prompt becomes tool-aware (READ-ONLY-mode rule etc.).
- 2026-01-05 `c6fc08453` (#454) — `custom-tools` + hook tool-wrapper subsystem (`packages/coding-agent/src/core/custom-tools/*`, `--tool`) merged into unified extensions (`registerTool`).
- 2026-03-22 `235b247f1` — "built-in tools work like extension tools" (definitions + renderers, snippets moved into tool definitions).
- 2026-08-24 `80e62761f` (#8512) — `powershell` built-in (Windows only).
- 2026-09-29 `8562bcf66` — codemode, tool_search, MCP as replaceable built-in extensions; "The core only gets general mechanisms; codemode, tool_search and MCP are built-in extensions that use them." Defaults still 4 tools.
- 2026-10-01 `7fd478a2e` — old `packages/agent/src/harness/tools/*` moved to `packages/durable/src/tools/*`.
- 2026-10-05 `04b97ef00` — `--tools` keeps MCP tools, adds patterns and `--no-mcp`.

## Evidence commits
`ffc9be886`, `60e4fcf01`, `186169a82`, `8f5523ed5`, `c6fc08453`, `235b247f1`, `80e62761f`, `8562bcf66`, `7fd478a2e`, `04b97ef00`

## Quirks
- grep/find/ls never became default in ~11 months; the model is expected to `rg` via bash. rg/fd are still auto-downloaded and prepended to bash PATH ([[pi--search-tools]]).
- Removed tools (history): `packages/mom/src/tools/*` (Slack bot, removed `0ed0d4343` 2026-04-30); web-ui tools artifacts/javascript-repl/extract-document (`b141e1fa2` 2026-05-20); browser-extension tools (`aa005d062` 2025-10-06, moved to sitegeist); demo `calculate`/`get-current-time` (`a055fd448` 2025-12-28); `packages/agent/src/harness/tools/*` + `experimental/micro/tools.ts` (`7fd478a2e`).
- Absent tools by stance: no todo ([[no-todo-tool]]), no web search/fetch ([[no-web-tools]]), no subagent/task ([[no-subagents-core]]), no plan mode ([[no-plan-mode]]), no background bash ([[no-background-bash]]).
- Three hard-coded copies of the default list could drift (unverified whether any divergence ever happened).

## Durable variant (packages/durable)
- `CodingTools` extension = exactly read, write, edit, bash; "Nothing installs it automatically"; `createPowerShellTool()` adds powershell (`packages/durable/src/tools/index.ts:21-25`). No grep/find/ls.

## Failures
- [[tool-allowlist-hides-mcp-tools]]
