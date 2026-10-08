---
type: failure
concepts: [auxiliary-model-calls]
harnesses: [opencode]
---
**Symptom** — The per-turn summary shown in place of the agent's last message hid the agent's final question or instruction to the user (e.g. "Now please run the command and paste the console output"), so the user never saw what was asked of them.

**Fix · [[opencode]]** `98fd53fd5f` 2025-12-29 "preserve imperative statements in summary": `summary.txt` now says "If the conversation ends with an unanswered question to the user, preserve that exact question … always include that exact request" (`packages/opencode/src/agent/prompt/summary.txt:1-11`). The LLM summary caller was later removed (`71a7ad1a4e` 2026-01-12); `SessionSummary.summarize` computes diffs only (`packages/opencode/src/session/summary.ts:102-127`).

**Lesson** — A summary that replaces the visible last message must carry pending asks verbatim.

Related: [[auxiliary-model-calls]] · [[summary-template-drops-goals]] · [[opencode--auxiliary-model-calls|opencode]]
