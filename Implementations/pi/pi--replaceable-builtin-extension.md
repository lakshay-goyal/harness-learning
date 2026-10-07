---
type: implementation
harness: pi
concept: replaceable-builtin-extension
commit: b30a6dd77
files: [packages/coding-agent/src/extensions/index.ts:7-13, packages/coding-agent/src/core/extensions/types.ts:2017-2049, packages/coding-agent/src/core/resource-loader.ts:116-150, packages/coding-agent/docs/settings.md:160, packages/coding-agent/docs/sdk.md:116, CONTRIBUTING.md:7-11, packages/coding-agent/README.md:19]
---
[[replaceable-builtin-extension]] in [[pi]].

## Mechanism
- Governance: "**pi's core is minimal**. If your feature does not belong in the core, it should be an extension. PRs that bloat the core will likely be rejected." (`CONTRIBUTING.md:7-9`); even hook points "should be well considered … to avoid adding unmaintainable bloat" (`CONTRIBUTING.md:11`).
- Headline: "Pi ships with powerful defaults but skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package that does it your way." (`README.md:19`, `packages/coding-agent/README.md:19`; wording from `de7e675de` 2026-10-02). "Pi is a minimal, extensible agent harness" (`packages/coding-agent/README.md:15`).
- Built-in list `builtInExtensions` (`packages/coding-agent/src/extensions/index.ts:7-13`):
  - `{name:"llama.cpp", builtin:true}` — provider only, registers no tools;
  - `{name:"codemode", replaceable:true, builtin:true}`, `{name:"tool-search", replaceable:true, builtin:true}`, `{name:"mcp", replaceable:true, builtin:true}` — "an extension that registers `codemode`, `tool_search`, or `/mcp` … takes over instead of running alongside the built-in one".
- `InlineExtension` options (`core/extensions/types.ts:2017-2049`): `name`, `factory`, `hidden`, `replaceable` ("Leave this extension out when another extension registers a tool, command, or flag with a name it registers during loading, instead of reporting a conflict… The factory still runs, so it should only register tools, commands, flags, and event handlers"), `builtin` ("`builtin:<name>` is an extension resource like a file: it loads by default, `pi config` lists it, `-builtin:<name>` … and `--no-extensions` disable it, and `-e builtin:<name>` loads it explicitly. … loads after project trust is resolved, so it cannot handle `project_trust`").
- Replacement algorithm `omitReplacedExtensions` (`core/resource-loader.ts:116-150`): collect `tool:`/`command:`/`flag:` names of all non-replaceable extensions; drop any replaceable extension sharing a name; for built-ins push warning "Extension X registers … so built-in extension `mcp` was not loaded… We recommend only having one or the other loaded at a time."
- Settings: `builtin:mcp|llama.cpp|codemode|tool-search` in `extensions`; `-builtin:mcp` disables; project `+builtin:`/`-builtin:` overrides user (`docs/settings.md:160`). MCP also `--no-mcp` (`docs/mcp.md:228,266`).
- SDK: built-ins NOT auto-loaded — add `createCodemodeExtension()`, `createToolSearchExtension()`, `createMcpExtension()` to `DefaultResourceLoader.extensionFactories` (`docs/sdk.md:116`; example `examples/sdk/14-codemode-mcp.ts`) → [[pi--sdk-embedding]].
- Design sentence of the MCP landing: "The core only gets general mechanisms; codemode, tool_search and MCP are built-in extensions that use them. Other extensions can replace these built-ins." (`8562bcf66`, closes #10040).
- Everything else "missing" is an example plugin, not core: sub-agents (`examples/extensions/subagent/`), plan mode (`plan-mode/`), approvals (`permission-gate.ts`, `protected-paths.ts`), sandbox (`sandbox/`, `gondolin/`), todos (`todo.ts`), checkpoints (`git-checkpoint.ts`) — full table in [[pi--extension-event-hooks]].
- **Provenance labels** (`core/source-info.ts:3-57`): every extension, command, skill, prompt template and tool carries `SourceInfo {path, source, scope: user|project|temporary, origin: package|top-level, baseDir?}`. Paths that name no file are synthetic: `builtin:<name>` (`BUILTIN_PATH_PREFIX`, `:15`) or angle-bracket `<inline:name>` (`:21-29`). Consumers: `/bug` bundle lists extensions by redacted source/scope/origin (`core/bug-report.ts:130-132`); crash log attributes stack frames to extensions (`core/crash-log.ts:47`) → [[pi--install-telemetry|install-telemetry]].

## Constants
| name | value | path:line |
|---|---|---|
| built-in extensions | 4 (`llama.cpp`, `codemode`, `tool-search`, `mcp`) | `src/extensions/index.ts:7-13` |
| replaceable built-ins | 3 | `src/extensions/index.ts:11-13` |

## Evolution
- 2025-11-12 per-feature "pi does not and will not …" README sections: sub-agents `e9935beb5`, background bash `271810c80`, to-dos `9066f58ca`, YOLO `b172beb92`, MCP `60e4fcf01` ("A 225-token README beats a 13,000-token MCP server description").
- 2025-12-17 `3424550d2` README "Philosophy" manifesto (No MCP / sub-agents / permission popups — "Security theater" / plan mode / to-dos / background bash).
- Removed packages, shrinking the monorepo toward the core: `aa005d062` 2025-10-06 browser-extension ("migrated to separate sitegeist repo"); `92bad8619` 2025-11-10 agent-old; `0f98decf6` 2025-12-28 proxy; `c6fc08453` 2026-01-05 hooks + custom-tools subsystems merged into extensions; `0ed0d4343` 2026-04-30 mom (Slack bot) + pods ("check out pi-chat"); `fe66edd94` 2026-04-30 Gemini CLI + Antigravity providers (reason unverified); `b141e1fa2` 2026-05-20 web-ui workspace; `b70c0f5b4` 2026-08-02 reverted switchable terminal renderers (#7473); `7fd478a2e` 2026-10-01 experimental harness in pi-agent-core + `packages/session-backends` + mini/micro frontends ("Durable sessions live in @earendil-works/pi-durable", 105k lines). Example removals: `c5c515f56` chalk-logger ("breaks TUI by using console.log directly"), `866d21c25` pi-dosbox (moved out), `9e05370b2` kimi-deferred-tools.
- 2026-09-22 `25cc5c7bf` Philosophy section dropped in docs refresh (#9898) — one week before MCP landed.
- 2026-09-29 `8562bcf66` codemode + MCP + tool_search as replaceable built-ins (v0.99.0, `packages/coding-agent/CHANGELOG.md:248`); `4259686d9` built-ins resolved as `builtin:<name>` resources (#10159).
- No built-in agent tool has ever been deleted (`git log --diff-filter=D` on tools dirs shows only the hooks/custom-tools merge).

## Evidence commits
`e9935beb5` `271810c80` `9066f58ca` `b172beb92` `60e4fcf01` `3424550d2` `25cc5c7bf` `8562bcf66` `4259686d9` `de7e675de` `0ed0d4343` `b141e1fa2` `7fd478a2e`

## Quirks
- Reversal: MCP absent (2025-11 → 2026-09-28) → built-in (2026-09-29) — "why" not stated; inferred: codemode made MCP cheap in context (unverified) → [[no-builtin-mcp-reversed]].
- `replaceable` factories still run before being dropped, so they must be side-effect free at load (`types.ts` doc comment).
- Built-ins cannot participate in `project_trust` (load after trust).

## Failures
- none specific.
