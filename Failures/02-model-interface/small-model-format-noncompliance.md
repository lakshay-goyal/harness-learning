---
type: failure
concepts: [auxiliary-model-calls]
harnesses: [opencode]
---
**Symptom** — Small title models emitted multi-line, quoted, prefixed or `<think>`-wrapped output, monotonous "Analyzing …" titles, or answered the user instead of titling.

**Fix · [[opencode]]**
- Prompt rules: `137e964131` 2025-06-21 "EXACTLY one line with NO line breaks"; `70229b150c` 2025-07-16 "needs to change due to small models"; `e9826e8a22` 2025-08-31 "You are a title generator. You output ONLY a thread title."; `fe57d7bb38` 2026-01-07 removed the "-ing verbs" rule that produced "Analyzing …", added "Vary your phrasing", temperature 0.5 (`packages/opencode/src/agent/agent.ts:240`).
- Code-side cleanup: `ad8a4bc744` 2025-07-26 strip `<think>` blocks; first non-empty line; cap 100 chars (`packages/opencode/src/session/prompt.ts:243-249`).
- Options isolation: `554572bc39` 2026-01-04 main-model thinking variant no longer applied to the small model; `3b72857124` 2025-11-19 / `bd9d7b3221` 2026-02-02 low reasoning effort for OpenAI title calls.

**Lesson** — Never trust a side-call's format: constrain in the prompt, then post-process in code, and give the side call its own sampling and reasoning options.

Related: [[auxiliary-model-calls]] · [[summarizer-continues-conversation]] · [[imperative-guideline-over-compliance]] · [[opencode--auxiliary-model-calls|opencode]]
