---
type: failure
concepts: [structured-compaction-summary, auxiliary-model-calls]
harnesses: [opencode]
---
**Symptom** — In non-English sessions the compaction summary and the generated session title came back in English; after compaction the agent continued in the wrong language because its only memory was the English summary.

**Root cause** — Side-call prompts (summarizer, title generator) are written in English and the model follows the prompt's language, not the conversation's, unless told otherwise.

**Fix · [[opencode]]**
- `fd77d31b49` 2026-01-21 (#9847) "change prompt to have the response with user language" (session title).
- `5daf2fa7f0` 2026-04-02 (#20581) "compaction agent responds in same language as conversation"; HEAD `compaction.txt`: "Respond in the same language as the conversation." (`packages/opencode/src/agent/prompt/compaction.txt:5`), mirrored in the v2 compaction agent prompt (`packages/core/src/plugin/agent.ts:37`).
- Gap: the v2 compaction request has no `system` field — only `model`, `http`, one user message, `tools: []`, `maxTokens` (`packages/core/src/session/compaction.ts:201-210`) — and `packages/core/src/session/compaction.ts` contains no language rule, so v2 summaries lose the fix (confirmed in code; runtime effect untested).

**Lesson** — Every side-call prompt over user content needs an explicit "answer in the conversation's language" rule, and it must travel with whichever prompt actually reaches the model.

Related: [[structured-compaction-summary]] · [[auxiliary-model-calls]] · [[transcript-serialization-for-summary]] · [[opencode--structured-compaction-summary|opencode impl]]
