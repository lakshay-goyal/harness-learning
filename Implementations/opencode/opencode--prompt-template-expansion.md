---
type: implementation
harness: opencode
concept: prompt-template-expansion
commit: ecc4916b5a
files: [packages/opencode/src/command/index.ts:10-11, packages/opencode/src/command/index.ts:42, packages/opencode/src/command/index.ts:47-48, packages/opencode/src/command/index.ts:81-86, packages/opencode/src/command/index.ts:105-140, packages/opencode/src/config/markdown.ts:5-6, packages/opencode/src/session/prompt.ts:160, packages/opencode/src/session/prompt.ts:1383-1408, packages/opencode/src/session/prompt.ts:1595, specs/v2/config.md:43-50, packages/opencode/src/command/template/review.txt:1-45, packages/core/src/plugin/command/review.txt]
---
[[prompt-template-expansion]] in [[opencode]].

## Mechanism
### Legacy runtime
- Sources: markdown command files in config dirs, config `command` entries, built-ins `/init` (AGENTS.md generator) and `/review` ("review changes [commit|branch|pr], defaults to uncommitted", runs as `subtask: true`), MCP server prompts (`source: "mcp"`), and skills (`source: "skill"`) (`packages/opencode/src/command/index.ts:10-11,47-48,81-86,105-140`) → [[skill-progressive-disclosure]], [[mcp-integration]].
- Substitution order (`packages/opencode/src/session/prompt.ts:1383-1408`): positional `$1…$N` (`/\$(\d+)/g`, `:1595`; the last placeholder swallows the remaining args) → `$ARGUMENTS` → if no placeholder at all, arguments appended after a blank line → `` !`cmd` `` spans run with the preferred shell, **no permission check, no tool hooks**, replaced by stdout → `@file` references resolved (`config/markdown.ts:5-6`; `prompt.ts:160`) → [[mention-expansion]].
- Because args are substituted before shell expansion, user-typed `` !`…` `` inside arguments also executes (observed from code; input is user-typed).
- Per-command `model`, `agent` and `subtask` (run in a subagent, then a synthetic "Summarize the task tool output above and continue with your task.").
- Built-in `/review` template (`packages/opencode/src/command/template/review.txt`, 101 lines): "You are a code reviewer"; `$ARGUMENTS` dispatch table (none → `git diff` + `git diff --cached` + `git status --short`; hash → `git show`; branch → `git diff $ARGUMENTS...HEAD`; PR → `gh pr view`/`gh pr diff`); "Diffs alone are not enough" — read whole modified files (`review.txt:1-45`).
### v2 runtime
- User-authored commands deliberately not ported: "Skills should cover named reusable prompt workflows"; legacy per-command `model`, `agent`, `subtask`, prompt shell expansion are dropped (`specs/v2/config.md:43-50`). Exception: built-in plugin commands survive — `packages/core/src/plugin/command.ts:7` ships a copy of the review template (`packages/core/src/plugin/command/review.txt`) that differs from legacy only by dropping "Don't flag style preferences…" and the "Great job / Thanks for" anti-flattery examples.

## Constants
| name | value | path:line |
|---|---|---|
| shell span regex | `` /!`([^`]+)`/g `` | `packages/opencode/src/config/markdown.ts:6` |

## Evolution
- 2025-12-21 `4f73d58031` improved built-in `/review` prompt; 2026-02-10 `60bdb6e9ba` flag behaviour changes explicitly.
- 2025-12-24 `2730e0c9cd` `/init` "Make it about 150 lines long" → 2026-04-01 `897d83c589` "only what an agent would miss" → [[generic-context-file-bloat]].

## Quirks / drift
- Template shell expansion bypasses the `bash` permission path and `tool.execute.before` hooks → [[side-door-input-bypasses-hooks]] (07-safety owner).

Contrast: pi substitutes bash-style args non-recursively and has no shell or `@file` expansion in templates → [[pi--prompt-template-expansion|pi]].
