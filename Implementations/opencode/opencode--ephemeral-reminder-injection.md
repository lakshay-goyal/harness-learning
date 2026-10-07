---
type: implementation
harness: opencode
concept: ephemeral-reminder-injection
commit: ecc4916b5a
files: [packages/opencode/src/session/reminders.ts:15-90, packages/opencode/src/session/prompt.ts:1092, packages/opencode/src/session/prompt.ts:1178-1184, packages/opencode/src/session/prompt.ts:1281, packages/opencode/src/session/prompt/plan.txt, packages/opencode/src/session/prompt/build-switch.txt, packages/opencode/src/session/prompt/plan-mode.txt, packages/opencode/src/tool/read.ts:356, packages/tui/src/component/prompt/index.tsx:131-138, packages/tui/src/component/prompt/move.tsx:15, packages/core/src/session/runner/max-steps.ts:1-4]
---
[[ephemeral-reminder-injection]] in [[opencode]].

## Mechanism
### Legacy runtime: producers
- `SessionReminders.apply({messages, agent, session})` runs each step after history is reloaded from the store (`packages/opencode/src/session/prompt.ts:1092,1180`).
- **Default mode** (`!experimentalPlanMode`): agent `plan` → push `plan.txt` (`<system-reminder># Plan Mode - System Reminder…`) as a `synthetic: true` text part onto the **last user message, in memory only**; any earlier assistant from `plan` + current agent `build` → push `build-switch.txt` ("Your operational mode has changed from plan to build. You are no longer in read-only mode…") (`packages/opencode/src/session/reminders.ts:26-48`).
- **Experimental plan mode**: on the transition step only, `sessions.updatePart` **persists** the part: build-switch + "A plan file exists at <path>. You should execute on the plan defined within it", or `plan-mode.txt` with `${planInfo}` = plan-file path (`reminders.ts:51-89`) → [[plan-mode]].
- Tool-result channel: read appends unseen nested AGENTS.md as `<system-reminder>` (`packages/opencode/src/tool/read.ts:356`) → [[context-file-hierarchy]].
- Client channel (TUI): "Note: The user opened the file … This may or may not be relevant to the current task" (`packages/tui/src/component/prompt/index.tsx:131-138`); "The user has changed the current working directory to …" (`packages/tui/src/component/prompt/move.tsx:15`).
- Last-step wrap-up: `MAX_STEPS_PROMPT` ("CRITICAL - MAXIMUM STEPS REACHED … Tools are disabled until next user input") appended as a trailing **assistant** message, not a reminder part (`packages/opencode/src/session/prompt.ts:1281`; text `packages/core/src/session/runner/max-steps.ts:1-4`) → [[step-budget-limit]].
### Legacy runtime: authority framing (system prompts)
- anthropic.txt:75 "bear no direct relation to the specific tool results or user messages in which they appear"; default.txt:78 similar; kimi.txt:17 "authoritative system directives that you MUST follow … restricting you to read-only actions during plan mode"; gpt-astra.txt:5 "harness instructions, not user-authored content. Read and follow them."; gpt.txt and codex.txt never mention the tag → [[per-model-system-prompt]].
### v2 runtime / origin/v2
- Base `system.txt`: "`<system-reminder>` blocks are harness instructions, not user-authored content. Read and follow them." (`origin/v2:packages/core/src/session/runner/prompt/system.txt:5`).
- Instruction changes are delivered as transcript updates ("These instructions replace all previously loaded ambient instructions.", `packages/core/src/instruction-context.ts:35-37`) → [[transcript-carried-system-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- 2025-07-09 `a826936702` plan reminder "you MUST NOT make any edits"; 2025-08-11 `457386ad08` + "Bash tool must only run readonly commands"; 2025-08-16 `ca3769b7fa` "CRITICAL: Plan mode ACTIVE … STRICTLY FORBIDDEN … ZERO exceptions" (shell still allowed by permissions).
- 2025-08-21 `9231043eb4` plan → build transition prompt; 2025-09-09 `bdc0f7c86d` build-switch wrapped in `<system-reminder>`, "run shell commands" added.
- 2025-10-25 `795b845782` anthropic.txt authority wording changed.
- 2026-01-13 `0a3c72d678` experimental plan mode with persisted reminders.
- 2026-04-27 `fab1768826` TUI file-context reminders.

## Quirks / drift
- In-memory re-attachment to "the last user message" moves the reminder when a new user message arrives, so the earlier user message changes bytes between requests (inferred cache cost; unmeasured).
- plan-mode.txt tells the plan agent to "Launch general agent(s)" / "at least 1 Plan agent" while its permissions deny `task: general` and no "Plan" subagent exists → [[prompt-names-unavailable-tools]].
- `MAX_STEPS_PROMPT` says "Tools are disabled" but the legacy runtime still sends tools on the last step → [[tool-description-drifts-from-implementation]].

Contrast: pi appends prompt-delta entries to the transcript instead of tagging user messages → [[pi--transcript-carried-system-prompt|pi]].
