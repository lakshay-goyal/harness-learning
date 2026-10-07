---
type: failure
concepts: [ask-user-tool]
harnesses: [opencode]
---
**Symptom** — In the question tool, the model added its own "Other" option next to the "Type your own answer" option the UI adds automatically.

**Root cause** — The model did not know what the client renders on its own.

**Fix · [[opencode]]** — `5092b5f07b` 2026-01-15 "clarify question tool guidance": "When `custom` is enabled (default), a 'Type your own answer' option is added automatically; don't include 'Other' or catch-all options" (`packages/opencode/src/tool/question.txt:8`).

**Lesson** — Tell the model what the UI adds so it doesn't duplicate it.

Related: [[ask-user-tool]] · [[tool-description-design]] · [[opencode--ask-user-tool|opencode]]
