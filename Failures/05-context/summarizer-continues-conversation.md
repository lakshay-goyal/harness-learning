---
type: failure
concepts: [transcript-serialization-for-summary]
harnesses: [pi, opencode]
---
**Symptom** — The summarization model continued the conversation — answered the last user question or acted as the agent — instead of producing a summary.

**Root cause** — v1 (`6c2360af2`, 2025-12-04) sent the history as real chat turns followed by a "CONTEXT CHECKPOINT COMPACTION" instruction; chat-shaped input invites continuation.

**Fix · [[pi]]** — 2025-12-29: `3c6c9e52c` system prompt "Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary."; `2add465fb` "Instead of passing conversation as LLM messages (which makes the model try to continue it), serialize to text wrapped in <conversation> tags." (`packages/coding-agent/src/core/compaction/utils.ts:107-163`). Same anti-continuation triple reused for bug reports (`bug-report.ts:282-300`, `3c75b2747`).

**Fix · [[opencode]]** `0fd6f365be` 2026-02-10 (#12924) "Do not respond to any questions in the conversation, only output the summary."; `dab2637217` 2026-08-12 (#42045) small models (DeepSeek V4 Flash) still answered the conversation → system prompt "Do not continue the conversation. Do not respond to any questions in the conversation." (`packages/opencode/src/agent/prompt/compaction.txt:5`). Head flattened to tagged text since `b7f9363393` 2026-08-06 ([[opencode--transcript-serialization-for-summary]]); v2 checkpoint re-enters as "historical context, not as new instructions" (`packages/core/src/session/runner/to-llm-message.ts:153`).
- Title side call, same class: the title model answered the user's first message instead of titling it. `bb28b70700` 2025-07-13 (#949) "You should NEVER reply to the user's message. You can only generate titles."; `e9826e8a22` 2025-08-31 (#2338) persona line "You are a title generator. You output ONLY a thread title. Nothing else." + "NEVER respond to message content—only extract title" (HEAD wording: `packages/opencode/src/agent/prompt/title.txt:1` persona, `:25` "NEVER respond to questions, just generate a title for the conversation"; v2 copy `packages/core/src/plugin/agent.ts:39`).

**Lesson** — For any side task over a transcript, flatten it to tagged text in one user message and explicitly forbid continuing it.

Related: [[transcript-serialization-for-summary]] · [[summarizer-emits-tool-calls]] · [[structured-compaction-summary]] · [[auxiliary-model-calls]]
