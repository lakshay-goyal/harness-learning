---
type: implementation
harness: opencode
concept: agent-profiles
commit: ecc4916b5a
files: [packages/opencode/src/agent/agent.ts:119-293, packages/opencode/src/agent/agent.ts:313-325, packages/opencode/src/agent/agent.ts:366-415, packages/core/src/v1/config/agent.ts:15-40, packages/opencode/src/config/agent.ts:13, packages/opencode/src/cli/cmd/agent.ts:125-175, packages/core/src/plugin/agent.ts:97-160, specs/v2/config.md:260-267]
---
[[agent-profiles]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Info** = name, description, mode (`primary|subagent|all`), native, hidden, model, variant, prompt, temperature, topP, color, steps, options, permission (`packages/opencode/src/agent/agent.ts:141-265`; schema `packages/core/src/v1/config/agent.ts:15-40`).
- **Native agents** (`packages/opencode/src/agent/agent.ts:140-265`):
  - `build` (primary, default): defaults + `question`, `plan_enter` allow.
  - `plan` (primary): edit denied except plan files, `task: {general: deny}` ([[plan-mode]]).
  - `general` (subagent): defaults + `todowrite: deny`; "Use this agent to execute multiple units of work in parallel".
  - `explore` (subagent): `*: deny`, then grep/glob/list/bash/webfetch/websearch/read allow, read-only external dirs; own prompt `explore.txt`; caller asked for "quick"/"medium"/"very thorough".
  - `compaction`, `title` (temperature 0.5), `summary`: hidden primaries with `*: deny` ([[auxiliary-model-calls]]).
- **Permission layering**: `Permission.merge(defaults, agent-specific, user)`; last match wins, so `permission` in user config overrides built-in agent restrictions ([[permission-ruleset]]).
- **Config** `agent.<name>` (`packages/opencode/src/agent/agent.ts:267-293`): `disable` deletes a built-in; unknown names create a new agent (mode `all`, permission `defaults + user`); fields override model/variant/prompt/description/temperature/top_p/mode/color/hidden/name/steps/options; agent `permission` appended last. Markdown agents load from `{agent,agents}/**/*.md` in config dirs (`packages/opencode/src/config/agent.ts:13`). Legacy `mode.<name>` entries fold into `agent` as primary (`packages/opencode/src/config/config.ts:549-557`).
- **Ordering**: list sorted with `default_agent` (or `build`) first, then by name (`packages/opencode/src/agent/agent.ts:313-325`).
- **Generation**: `opencode agent create` → `Agent.generate({description, model})`: system `generate.txt` (Claude Code "elite AI agent architect" copy), user message lists existing identifiers that "must NOT be used", "Return ONLY the JSON object", `temperature: 0.3`, structured output; OpenAI OAuth path drops system messages (`packages/opencode/src/agent/agent.ts:366-415`; CLI `packages/opencode/src/cli/cmd/agent.ts:125-175` then prompts for mode and tools).
- **Step cap**: `agent.steps ?? Infinity`; last step injects `MAX_STEPS_PROMPT` as an assistant prefill (`packages/opencode/src/session/prompt.ts:1178`, `packages/opencode/src/session/prompt.ts:1281`) ([[step-budget-limit]]).

### v2 runtime
- Built-in agents are declared by a core plugin via `ctx.agent.transform` (`packages/core/src/plugin/agent.ts:97-160`): `build` (default ID), `plan`, `general`, … with `permissions` arrays.
- Config: agent `prompt` renamed `system`; no agent `temperature`/`top_p`; `description`, `hidden`, `steps`, `color`, `model` + `variant` kept (`specs/v2/config.md:260-267`).

## Constants
| name | value | path:line |
|---|---|---|
| `title` agent temperature | 0.5 | `packages/opencode/src/agent/agent.ts:240` |
| generate temperature | 0.3 | `packages/opencode/src/agent/agent.ts:396` |
| custom agent default mode | `all` | `packages/opencode/src/agent/agent.ts:276` |

## Evolution
- 2025-07-09 `a826936702` "modes concept" (build/plan modes, later folded into agents).
- 2025-12-14 `fed4776451` explore prompt moved from inline to `explore.txt`.
- 2026-06-10 `3ad6923c61` plan denies `task: general`.
- 2026-06-03 `76ee87ead8` v2 runtime with core agent definitions.

## Quirks / drift
- Native agents' permission arrays include the user layer, then config `agent.<name>.permission` is appended again (`packages/opencode/src/agent/agent.ts:293`), so agent-level config beats global config.
- The generator prompt still mentions "CLAUDE.md files" (`packages/opencode/src/agent/generate.txt`).

pi contrast: no profiles in core; the subagent example reads role files with `tools`/`model` frontmatter and spawns the CLI ([[pi--subagent-as-subprocess|pi]]).
