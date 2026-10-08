---
type: concept
stage: tools
tier: must-have
aliases: [question tool, Question.Service, AskUserQuestion, "(Recommended)", "Type your own answer", request_user_input_async, send_message_to_user_async, ask-user tool, multiple-choice question tool, request_user_input, structured-user-question-tool]
harnesses: [opencode, codex]
---
A blocking tool that asks the human structured questions mid-turn and returns the answers as its result.

## Why
- Free-text questions end the turn; the model loses its in-flight plan and the user must re-establish context.
- Structured options with a recommended default let the user answer in one keystroke and keep the turn going.
- Headless runs cannot answer, so the tool must be gated by client type.
- Free-text questions in the assistant message end the turn and lose structure; a tool call keeps the turn open and returns machine-mappable answers (ids → choices).
- Unconstrained question-asking stalls autonomous flows: models asked in chat instead of using permission parameters, or used the question tool to ask for permissions ([[guideline-softening]], codex M5a `8dcbd29edd`).
- Must not run concurrently with other tools or inside scripts — a human can answer one prompt at a time ([[interactive-tool-in-parallel-batch]]).
- Answers are user-authored: a downstream LLM approval reviewer can treat them as trusted provenance ([[llm-approval-reviewer]]).

## Design space
- **Multiple-choice questions with an automatic custom-answer option** (opencode) vs free-text only.
- Recommended option first with a "(Recommended)" label (opencode prompt rule).
- Gating: only interactive clients (opencode `app`/`cli`/`desktop`) and per-agent permission (opencode: denied by default, allowed for build/plan).
- Dismissal: stop the loop (opencode `RejectedError`) vs return "unanswered" and continue.
- Parallel batches: interactive tools must not run concurrently with others → [[interactive-tool-in-parallel-batch]].
- Example extension only, not core (pi `question.ts`/`questionnaire.ts`, [[pi--plugin-tools]]).
- **No question tool; ask in prose and end the turn** (pi).
- **Blocking multiple-choice tool, mode-scoped** (codex `request_user_input`: availability text and runtime check driven by one mode policy) vs always available.
- Option shape: 2–3 mutually exclusive choices, recommended first with "(Recommended)", client-added "Other" (codex) vs free text only.
- Non-blocking variants that post a question/message and continue (codex `request_user_input_async`, `send_message_to_user_async`, root agent only).
- Exposure: direct-only, never callable from code-mode scripts (codex `DirectModelOnly`) → [[code-mode]].
- Client-channel failure: fatal turn error (codex) vs tool error result → [[tool-error-as-result]].
- Elicitation time excluded from tool timeouts → [[elicitation-pause]].
- (folded from `structured-user-question-tool`, codex framing) The model asks the user 1–3 short multiple-choice questions through a blocking tool call; the client renders them (adding a free-text "Other"), and the answers come back as the tool result.

## Implementations
- [[opencode--ask-user-tool|opencode]] — `question` tool; output `"q"="answers"`; registered only for interactive clients; denied in `opencode run`.
- [[codex--ask-user-tool|codex]] — `request_user_input(questions[{id, header, question, options[{label, description}]}])`, ≤3 questions, mode-scoped, `DirectModelOnly`, async variants for root agents.

## Failures
- [[excessive-permission-questions]]
- [[duplicate-catch-all-option]]
- [[tool-arg-shape-drift]]
- [[interactive-tool-in-parallel-batch]]
- (04) [[mode-state-confusion]] (Plan vs Default; "Hard interaction rule" softened `3dd9a37e0b`)

## Related
[[plan-mode]] · [[permission-ruleset]] · [[agent-profiles]] · [[tool-description-design]] · [[parallel-tool-execution]]
[[plan-mode]] · [[task-list-tool]] · [[elicitation-pause]] · [[parallel-tool-execution]] · [[tool-description-design]] · [[llm-approval-reviewer]] · [[model-requested-permissions]]
