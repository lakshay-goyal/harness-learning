---
type: implementation
harness: opencode
concept: ask-user-tool
commit: ecc4916b5a
files: [packages/opencode/src/tool/question.ts:11-38, packages/opencode/src/tool/question.txt:8-10, packages/opencode/src/tool/registry.ts:207, packages/opencode/src/tool/registry.ts:233, packages/opencode/src/agent/agent.ts:126, packages/opencode/src/agent/agent.ts:148, packages/opencode/src/agent/agent.ts:163, packages/opencode/src/cli/cmd/run.ts:434-445]
---
[[ask-user-tool]] in [[opencode]].

## Mechanism
- Tool `question` takes `questions[]` with options (`multiple`, `custom`); `question.ask` blocks until the UI answers; output `User has answered your questions: "q"="a, b". You can now continue…`, unanswered → `"Unanswered"` (`packages/opencode/src/tool/question.ts:24-36`).
- Registered only when `["app","cli","desktop"].includes(flags.client) || flags.enableQuestionTool` (`packages/opencode/src/tool/registry.ts:207,233`).
- Permission: default `question: "deny"`, allowed for `build` and `plan` (`packages/opencode/src/agent/agent.ts:126,148,163`); denied in headless `opencode run` (`packages/opencode/src/cli/cmd/run.ts:434-445`).
- Dismissal raises `Question.RejectedError`, which stops the loop; the transcript records "The user dismissed this question" (`packages/opencode/src/question/index.ts:29`).
- Description: recommended option first with "(Recommended)"; "a 'Type your own answer' option is added automatically; don't include 'Other' or catch-all options" (`packages/opencode/src/tool/question.txt:8-10`).
- v2 runtime: `QuestionTool` in core built-ins (`packages/core/src/tool/builtins.ts:40`).

## Constants
| name | value | path:line |
|---|---|---|
| clients | `app`, `cli`, `desktop` | `packages/opencode/src/tool/registry.ts:207` |

## Evolution
- 2026-01-07 `e37fd9c105` interactive question tool added.
- 2026-01-15 `5092b5f07b` "don't include Other" guidance → [[duplicate-catch-all-option]].
- 2026-01-24 `397ee419d1` `label`/`header` `.max(30)` moved from schema to description → [[tool-arg-shape-drift]].
- 2026-05-20 `26008696e1` schema decode failures surfaced as friendly tool errors.

## Quirks / drift
- Plan-mode prompt forbids using `question` to ask "Is this plan okay?" because `plan_exit` does that → [[plan-mode]].

Contrast: pi ships ask-user only as example extensions ([[pi--plugin-tools]]).
