---
type: concept
stage: tools
tier: candidate
aliases: [question tool, Question.Service, AskUserQuestion, "(Recommended)", "Type your own answer"]
harnesses: [opencode]
---
A blocking tool that asks the human structured questions mid-turn and returns the answers as its result.

## Why
- Free-text questions end the turn; the model loses its in-flight plan and the user must re-establish context.
- Structured options with a recommended default let the user answer in one keystroke and keep the turn going.
- Headless runs cannot answer, so the tool must be gated by client type.

## Design space
- **Multiple-choice questions with an automatic custom-answer option** (opencode) vs free-text only.
- Recommended option first with a "(Recommended)" label (opencode prompt rule).
- Gating: only interactive clients (opencode `app`/`cli`/`desktop`) and per-agent permission (opencode: denied by default, allowed for build/plan).
- Dismissal: stop the loop (opencode `RejectedError`) vs return "unanswered" and continue.
- Parallel batches: interactive tools must not run concurrently with others → [[interactive-tool-in-parallel-batch]].
- Example extension only, not core (pi `question.ts`/`questionnaire.ts`, [[pi--plugin-tools]]).

## Implementations
- [[opencode--ask-user-tool|opencode]] — `question` tool; output `"q"="answers"`; registered only for interactive clients; denied in `opencode run`.

## Failures
- [[excessive-permission-questions]]
- [[duplicate-catch-all-option]]
- [[tool-arg-shape-drift]]
- [[interactive-tool-in-parallel-batch]]

## Related
[[plan-mode]] · [[permission-ruleset]] · [[agent-profiles]] · [[tool-description-design]] · [[parallel-tool-execution]]
