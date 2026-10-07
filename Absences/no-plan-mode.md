---
type: absence
harnesses: [pi]
---
# no-plan-mode

**What's missing**
- No `/plan`, no read-only mode flag, no EnterPlanMode/ExitPlanMode tool, no loop-level "mode" concept.

**Evidence of decision**
- Pre-Dec-2025 README (removed lines in the `3424550d2` diff): "No Planning Mode — Tell the agent to think through problems without modifying files. For persistent plans, write to a file: PLAN.md".
- `3424550d2` (2025-12-17) Philosophy: "**No plan mode.** Gather context in one session, write plans to file, start fresh for implementation."
- `25cc5c7bf^:packages/coding-agent/README.md:543`: "**No plan mode.** Write plans to files, or build it with extensions, or install a package." The section was deleted in `25cc5c7bf` (2026-09-22).
- HEAD `README.md:19`: "…skips features like sub-agents and plan mode." Plan mode is still named at HEAD.
- The system-prompt rule "You are in READ-ONLY mode - you cannot modify files or execute arbitrary commands" (added `186169a82`) was **removed** in `e3dd4f21d` (2026-01-08). That commit introduced extension tool overrides and `setActiveTools`. Inferred: once extensions could supply tools, the absence of bash/edit/write no longer implied read-only.
- "Use bash ONLY for read-only operations (git log, gh issue view, curl, etc.) - do NOT modify any files" was silently removed in `b846a4bfc` (2026-01-20, #645). Inferred: a prompt-only read-only rule could not be enforced without a permission system.

**Opt-in replacement**
- `packages/coding-agent/examples/extensions/plan-mode/` ("Claude Code-style plan mode", `examples/extensions/README.md:51`):
  - Swaps the active tool set: `PLAN_MODE_TOOLS = ["read","bash","grep","find","ls","questionnaire"]` vs `NORMAL_MODE_TOOLS = ["read","bash","edit","write"]` (`plan-mode/index.ts:22-23`; `pi.setActiveTools` at `:108,:112`).
  - Filters bash through a regex allowlist (`DESTRUCTIVE_PATTERNS` / `SAFE_PATTERNS`, `plan-mode/utils.ts:7,44,97-99`), enforced in a `tool_call` block (`plan-mode/index.ts:168`) → [[tool-call-gate]].
  - Extracts the plan from a `Plan:` numbered list and tracks progress via `[DONE:n]` tags in assistant text (`plan-mode/utils.ts:154`; `plan-mode/README.md:9-11,34`).
- Durable runtime: also example-only, `packages/durable/test/examples/27-plan-mode.ts`.
- `preset.ts` example: named presets (model, thinking, tools, instructions) via `--preset` / `/preset`. This is a mode switch built without a mode concept.

**History**
- The plan-mode example was the driver for the hook API. `57bba4e32` / `059292ead` ("WIP: Add hook API for dynamic tool control with plan-mode hook example", 2026-01-03) were merged as `91fae8b2f` (2026-01-04) and enhanced in `e8f1322ee` (#694).
- Never reversed. Unlike MCP, plan mode has not been promoted to a built-in extension ([[no-builtin-mcp-reversed]]).

**Implication**
- Plan mode decomposes into "tool-set swap + bash allowlist + text-tag progress", all expressible with `setActiveTools` plus a `tool_call` block. The core needs no mode concept.
- The prompt-only "read-only" rules were deleted. A mode must be enforced by the toolset or a gate, not by prose → [[minimal-default-toolset]], [[dynamic-tool-guidelines]].
- Plans live in files (PLAN.md), which survive compaction and session switches. See [[no-todo-tool]].

**opencode contrast**: implements it: `plan` agent (edits denied except `.opencode/plans/*.md`) + `plan_exit` tool; the model-driven `plan_enter` was disabled (`packages/opencode/src/agent/agent.ts:156-180`; `fa559b0385`) — see [[plan-mode]] / [[plan-mode-vs-none]] / [[no-model-initiated-plan-entry]].

Related: [[tool-call-gate]] · [[extension-event-hooks]] · [[minimal-default-toolset]] · [[replaceable-builtin-extension]] · [[no-subagents-core]] · [[no-permission-prompts]] · [[Absences]]
