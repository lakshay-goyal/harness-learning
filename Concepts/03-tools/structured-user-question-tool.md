---
type: concept
stage: tool-design
tier: candidate
aliases: [request_user_input_async, send_message_to_user_async, ask-user tool, multiple-choice question tool, request_user_input]
harnesses: [codex]
---
The model asks the user 1–3 short multiple-choice questions through a blocking tool call; the client renders them (adding a free-text "Other"), and the answers come back as the tool result.

## Why
- Free-text questions in the assistant message end the turn and lose structure; a tool call keeps the turn open and returns machine-mappable answers (ids → choices).
- Unconstrained question-asking stalls autonomous flows: models asked in chat instead of using permission parameters, or used the question tool to ask for permissions ([[guideline-softening]], codex M5a `8dcbd29edd`).
- Must not run concurrently with other tools or inside scripts — a human can answer one prompt at a time ([[interactive-tool-in-parallel-batch]]).
- Answers are user-authored: a downstream LLM approval reviewer can treat them as trusted provenance ([[llm-approval-reviewer]]).

## Design space
- **No question tool; ask in prose and end the turn** (pi).
- **Blocking multiple-choice tool, mode-scoped** (codex `request_user_input`: availability text and runtime check driven by one mode policy) vs always available.
- Option shape: 2–3 mutually exclusive choices, recommended first with "(Recommended)", client-added "Other" (codex) vs free text only.
- Non-blocking variants that post a question/message and continue (codex `request_user_input_async`, `send_message_to_user_async`, root agent only).
- Exposure: direct-only, never callable from code-mode scripts (codex `DirectModelOnly`) → [[code-mode]].
- Client-channel failure: fatal turn error (codex) vs tool error result → [[tool-error-as-result]].
- Elicitation time excluded from tool timeouts → [[elicitation-pause]].

## Implementations
- [[codex--structured-user-question-tool|codex]] — `request_user_input(questions[{id, header, question, options[{label, description}]}])`, ≤3 questions, mode-scoped, `DirectModelOnly`, async variants for root agents.

## Failures
- (04) [[mode-state-confusion]] (Plan vs Default; "Hard interaction rule" softened `3dd9a37e0b`)

## Related
[[plan-mode]] · [[plan-checklist-tool]] · [[elicitation-pause]] · [[parallel-tool-execution]] · [[tool-description-design]] · [[llm-approval-reviewer]] · [[model-requested-permissions]]
