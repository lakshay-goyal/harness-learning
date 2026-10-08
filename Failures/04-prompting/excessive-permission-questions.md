---
type: failure
concepts: [per-model-system-prompt, ask-user-tool]
harnesses: [opencode]
---
**Symptom** — GPT codex models stopped mid-task to ask "Should I proceed?"-style permission questions.

**Root cause** — "Ask only when needed; suggest ideas; mirror the user's style." left "needed" to the model.

**Fix · [[opencode]]** — `5622c53e1f` 2026-01-20 "adjust codex prompt to discourage unnecessary question asking": enumerated reasons to ask (blocked with no safe default, destructive, missing secret), ask one question with a recommended default, and "Never ask permission questions like 'Should I proceed?' or 'Do you want me to run tests?'; proceed with the most reasonable option and mention what you did." (`packages/opencode/src/session/prompt/codex.txt:43-49`).

**Lesson** — Enumerate the only valid reasons to ask instead of saying "when needed".

Related: [[per-model-system-prompt]] · [[ask-user-tool]] · [[guideline-softening]] · [[opencode--per-model-system-prompt|opencode]]
