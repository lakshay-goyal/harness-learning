---
type: implementation
harness: pi
concept: system-prompt-override
commit: b30a6dd77
files: [packages/coding-agent/src/core/resource-loader.ts:1209-1235, packages/coding-agent/src/core/system-prompt.ts:9-37, packages/coding-agent/src/core/system-prompt.ts:152-173, packages/coding-agent/src/core/system-prompt.ts:199-205, packages/coding-agent/src/core/extensions/runner.ts:1420-1472, packages/coding-agent/src/core/extensions/types.ts:919-929, packages/coding-agent/src/core/extensions/types.ts:1466-1470, packages/coding-agent/src/core/agent-session.ts:1737-1801, packages/coding-agent/src/core/agent-session.ts:2058-2106]
---
[[system-prompt-override]] in [[pi]].

## Mechanism
Three tiers, from mildest to most opaque:

1. **Append** — `APPEND_SYSTEM.md` (project `.pi/` if trusted, else `~/.pi/agent/`) and repeatable `--append-system-prompt <text|path>` → joined `\n\n` into `appendSystemPrompt` → section `addendum`, placed after `docs` and **before** `project_context`/`skills`/`cwd` (`resource-loader.ts:1223-1235`; `agent-session.ts:1709-1710`; `system-prompt.ts:172`; `docs/cli.md:231`).
2. **Replace base** — `SYSTEM.md` (trusted project `.pi/SYSTEM.md` beats `~/.pi/agent/SYSTEM.md`; "Files with the same name are not combined", `docs/configuration.md:19-39`) or `--system-prompt <text|path>` → `customPrompt` → becomes `preamble`; replaces preamble+tools+rules+docs only; addendum, project context, skills, cwd and extension sections still appended (`system-prompt.ts:152-186`).
   - Project-local files honored only if `isProjectTrusted()` (`resource-loader.ts:1210-1212, 1224-1226`; trigger list `trust-manager.ts:30-39`) → [[project-trust-gate]]. SDK hooks `systemPromptOverride` / `appendSystemPromptOverride` (`resource-loader.ts:307-308`).
3. **Plugin hook `before_agent_start`** (`extensions/types.ts:919-929`, runner `extensions/runner.ts:1420-1472`), fired once per user prompt after input/template expansion (`agent-session.ts:2058-2065`):
   - Structured, chained mutation: `event.systemPromptOptions` is the mutable `NormalizedBuildSystemPromptOptions` (selectedTools, toolSnippets, toolGuidelines, promptGuidelines, appendSystemPrompt, `sections`, contextFiles, skills…); "Later handlers observe mutations made by earlier handlers" (`types.ts:927-928`); `event.systemPrompt` / `ctx.getSystemPrompt()` re-render from current options (`runner.ts:1426-1434, 1444-1446`).
   - Extra named sections: `sections[name]` must match `/^[a-z][a-z0-9_-]*$/`, `preamble` reserved (`system-prompt.ts:58, 144-148`); e.g. MCP writes `mcp_servers` here (`extensions/mcp/index.ts:1142-1148`).
   - Full override: return `{systemPrompt}` → `forceSystemPrompt` ("Replace the complete system prompt for this turn. Later handlers observe this exact override", `types.ts:1468-1469`; `runner.ts:1454-1456`). Handler may also inject a `custom` message (`types.ts:1467`; `agent-session.ts:2090-2100`).
   - Handler errors are reported, not fatal (`runner.ts:1458-1467`).
   - Tool selection: explicit edit of `selectedTools` wins; otherwise live loadout (`setActiveTools()`) authoritative (`agent-session.ts:2066-2073`).
- **How the result reaches the model**: options → `_preparePromptAndToolLoadout` → `diffSystemPromptSections` patch prepended to the run's messages (`agent-session.ts:1737-1749, 2104-2106`) → [[transcript-carried-system-prompt]]. Forced prompts are **not recorded**: `_installAgentForcedPromptProjection` collapses all system messages into one head with the forced text + current tools at request time (`agent-session.ts:1786-1801`; `16292398a`); the transcript keeps the structured sections.
- `buildSystemPromptState`: forced → `{content: forced}` with no sections; else `{content:"", sections}` (`system-prompt.ts:199-205`).

## Constants
| name | value | path:line |
|---|---|---|
| section name regex | `/^[a-z][a-z0-9_-]*$/` | `system-prompt.ts:58` |
| addendum position | after docs, before project_context | `system-prompt.ts:172-173` |
| file names | SYSTEM.md, APPEND_SYSTEM.md | `resource-loader.ts:1210, 1224` |

## Evolution
- `b1c2c32e2`/`a0fa25410` 2025-11-12: `--system-prompt` accepts file path; "Project context and datetime still appended automatically".
- `57146de20` 2025-12-28: `before_agent_start` hook ("prompt injection point").
- `c6fc08453` 2026-01-05 (#454): hooks + custom tools merged into `extensions/`.
- `b846a4bfc` 2026-01-20 (#645): ResourceLoader discovers SYSTEM.md / APPEND_SYSTEM.md.
- `4e919868f` 2026-04-22 (#3539): chain system prompt in `before_agent_start` — `ctx.getSystemPrompt()` had ignored earlier handlers' changes → [[prompt-hook-chain-sees-stale-prompt]].
- `e2fd651eb` 2026-05-16 (#4541): custom-prompt path got XML project-context boundaries first.
- `89a92207f` 2026-06-05 (#5332): project SYSTEM.md/APPEND_SYSTEM.md trust-gated.
- `e547bb9f4`, `fd6659dd5` 2026-06-30 (#6162): run prompt preserved across tool refresh in `prepareNextTurn` ("forced system prompt dropped").
- `3dd4623ee` 2026-08-10 (#7887): custom prompts concatenated cwd with later appended content → newline fix.
- `9e05370b2` 2026-09-16 (#9548): options become sections; hook gets `systemPromptOptions` with `sections`.
- `e4c75a732` 2026-09-16: forced prompt replaces instead of arriving as a later update on models with native mid-convo system messages → [[forced-system-prompt-applied-as-late-update]].
- `16292398a` 2026-09-17: "send forced system prompts without recording them" — removes `SystemMessage.replace`; commit body: persisting a forced prompt "wrote the full prompt and tool list on every change, lost the structured sections while forced, and disabled mid-conversation system messages for the rest of the session once a replacement existed".

## Evidence commits
`a0fa25410`, `57146de20`, `c6fc08453`, `b846a4bfc`, `4e919868f`, `e2fd651eb`, `89a92207f`, `e547bb9f4`, `fd6659dd5`, `3dd4623ee`, `9e05370b2`, `e4c75a732`, `16292398a`.

## Quirks
- `customPrompt` replaces even the tool list and rules — a SYSTEM.md author must restate tool guidance (tool declarations still sent via API).
- Example `pirate.ts` uses the full-override path by concatenating `event.systemPrompt + …` (`examples/extensions/pirate.ts:28-43`) — opts out of section patching/caching for that run; the structured path (`systemPromptOptions.sections`) would be cache-friendlier.
- Forced prompt affects cache: projection rewrites the head each run it is active (cache impact unverified).
- Evals use a hidden inline `before_agent_start` extension to strip `<docs>` for the without-docs arm and `verifySystemPrompt` fails closed if `<rules>` missing (`packages/evals/src/harness.ts:257-270, 289-301`; markers updated `1247476e6`, validated from transcript `5a3a03a7f`) → [[harness-evals]].

## Durable variant (packages/durable)
- No SYSTEM.md handling; prompt = ordered extension `sections` rendered per request; a section that throws keeps its previously shown text (`packages/durable/src/harness/prompt.ts:25-48`). Durable `pi-prompt` has no addendum (`experimental/durable/prompt.ts:20`).

## Failures
- [[forced-system-prompt-applied-as-late-update]]
- [[prompt-hook-chain-sees-stale-prompt]]
- [[markdown-boundaries-ingested-inconsistently]]
