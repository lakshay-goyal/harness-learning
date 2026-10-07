---
type: implementation
harness: opencode
concept: plan-mode
commit: ecc4916b5a
files: [packages/opencode/src/agent/agent.ts:156-180, packages/opencode/src/session/reminders.ts:24-50, packages/opencode/src/session/reminders.ts:70-89, packages/opencode/src/session/prompt/plan.txt:1-9, packages/opencode/src/tool/plan.ts:16-75, packages/opencode/src/tool/registry.ts:248, packages/opencode/src/session/prompt/plan-mode.txt:11-31, packages/core/src/plugin/agent.ts:132-149]
---
[[plan-mode]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Plan = an agent profile** ([[agent-profiles]]), description "Plan mode. Disallows all edit tools." Ruleset = defaults + `question: allow`, `plan_exit: allow`, `task: {general: deny}`, `external_directory: {<data>/plans/*: allow}`, `edit: {"*": deny, ".opencode/plans/*.md": allow, <data>/plans/*.md (worktree-relative): allow}` + user config (`packages/opencode/src/agent/agent.ts:156-180`). `edit` covers edit/write/apply_patch.
- **Shell is not restricted**: defaults' `*: allow` still applies to `bash`; read-only shell is prompt-only ([[read-only-mode-bypassed-via-shell]]).
- **Default reminder** (flag off): while the agent is `plan`, `plan.txt` is appended as a synthetic part to the last user message every turn: "CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. STRICTLY FORBIDDEN: ANY file edits … Do NOT use sed, tee, echo, cat, or ANY other bash command to manipulate files … ZERO exceptions." (`packages/opencode/src/session/reminders.ts:26-36`; `packages/opencode/src/session/prompt/plan.txt:1-9`) ([[ephemeral-reminder-injection]]).
- **Leaving plan**: if any earlier assistant message came from `plan` and the agent is now `build`, `build-switch.txt` is appended: "Your operational mode has changed from plan to build. You are no longer in read-only mode…" (`packages/opencode/src/session/reminders.ts:37-47`) ([[stale-mode-reminder-persists]]).
- **Experimental plan mode** (`OPENCODE_EXPERIMENTAL_PLAN_MODE` and client `cli`, `packages/opencode/src/tool/registry.ts:248`):
  - On entering, one injected `plan-mode.txt` reminder with the plan file path (`packages/opencode/src/session/reminders.ts:70-89`): phased workflow, "Launch up to 3 explore agents IN PARALLEL", "Launch general agent(s) to design", "Launch at least 1 Plan agent" (`packages/opencode/src/session/prompt/plan-mode.txt:11-31`).
  - `plan_exit` tool asks a Yes/No question "Plan at X is complete. Would you like to switch to the build agent…"; Yes → synthetic user message with `agent: "build"`: "The plan at X has been approved, you can now edit files. Execute the plan" (`packages/opencode/src/tool/plan.ts:16-75`).
- **Entry is user-only**: `plan_enter` tool removed from the registry (`fa559b0385`, 2026-02-24, "temporarily"); build still grants `plan_enter: allow` (`packages/opencode/src/agent/agent.ts:149`) and `plan-enter.txt` remains ([[model-initiated-mode-switch]]).
- **Delegation**: plan cannot call `general` (denied) but can call `explore`, which has `bash: allow` (`packages/opencode/src/agent/agent.ts:196-209`) ([[read-only-mode-bypass-via-subagent]]).

### v2 runtime
- `plan` agent in the core agent plugin: same edit denies and plan-file allows, `plan_exit` allowed; no `task` rule because v2 has no task tool yet (`packages/core/src/plugin/agent.ts:132-149`).

## Constants
| name | value | path:line |
|---|---|---|
| plan file glob | `.opencode/plans/*.md`, `<data>/plans/*.md` | `packages/opencode/src/agent/agent.ts:171-175` |

## Evolution
- 2025-07-09 `a826936702` "modes concept": plan prompt "you MUST NOT make any edits".
- 2025-08-11 `457386ad08` "fix plan mode bash tool making changes" → "Bash tool must only run readonly commands".
- 2025-08-16 `ca3769b7fa` capitals: "STRICTLY FORBIDDEN … Do NOT use sed, tee, echo, cat…".
- 2025-08-21 `9231043eb4` plan → build transition prompt; 2025-09-09 `bdc0f7c86d` wrapped in `<system-reminder>`.
- 2025-11-30 `a4eba2e6e9` responsibility section (delegate explore, ask questions).
- 2026-01-13 `0a3c72d678` plan_enter/plan_exit tools + experimental workflow (#8281).
- 2026-02-24 `fa559b0385` `plan_enter` disabled; 2026-04-10 `44f38193c0` tool internals moved to Effect, enter tool gone.
- 2026-05-09 `b8ca71d309` Plan Mode bypass via subagent fixed by inheriting agent denies; reverted 2026-06-10 `3ad6923c61` in favor of `task: {general: deny}`.

## Quirks / drift
- `plan-mode.txt` tells the plan agent to launch `general` and "Plan" agents, but `general` is denied to plan and no "Plan" subagent exists (task list excludes primary agents, `packages/opencode/src/tool/registry.ts:266`) ([[prompt-names-unavailable-tools]]).
- Test `packages/opencode/test/agent/plan-mode-subagent-bypass.test.ts` pins the bypass regression.

pi contrast: no plan mode; example extension only ([[no-plan-mode]]).
