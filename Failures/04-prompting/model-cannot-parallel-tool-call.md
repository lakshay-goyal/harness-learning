---
type: failure
concepts: [per-model-system-prompt, parallel-tool-execution]
harnesses: [opencode]
---
**Symptom** — Arcee Trinity mishandled multiple tool calls per message and re-ran the same search in loops (inferred from the rules added; commit bodies give no transcript).

**Root cause** — Parallel-call encouragement inherited from the fallback prompt did not fit this model.

**Fix · [[opencode]]** — `015cd404e4` 2026-02-03 Trinity prompt (reverted same day), `95ad6758af` 2026-02-06 re-added: "Use exactly one tool per assistant message. After each tool call, wait for the result before continuing." and "Avoid repeating the same tool with the same parameters once you have useful results" (`packages/opencode/src/session/prompt/trinity.txt:84-86`); examples rewritten to read files one at a time. Mechanical backstop for repetition → [[repeated-tool-call-detection]].

**Lesson** — Parallel-call encouragement must be model-conditional; GPT gets "parallelize", Trinity gets "one per message".

Related: [[per-model-system-prompt]] · [[parallel-tool-execution]] · [[repeated-tool-call-detection]] · [[opencode--per-model-system-prompt|opencode]]
