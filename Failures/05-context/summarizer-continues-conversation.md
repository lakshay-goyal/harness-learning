---
type: failure
concepts: [transcript-serialization-for-summary]
harnesses: [pi]
---
**Symptom** — The summarization model continued the conversation — answered the last user question or acted as the agent — instead of producing a summary.

**Root cause** — v1 (`6c2360af2`, 2025-12-04) sent the history as real chat turns followed by a "CONTEXT CHECKPOINT COMPACTION" instruction; chat-shaped input invites continuation.

**Fix · [[pi]]** — 2025-12-29: `3c6c9e52c` system prompt "Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary."; `2add465fb` "Instead of passing conversation as LLM messages (which makes the model try to continue it), serialize to text wrapped in <conversation> tags." (`packages/coding-agent/src/core/compaction/utils.ts:107-163`). Same anti-continuation triple reused for bug reports (`bug-report.ts:282-300`, `3c75b2747`).

**Lesson** — For any side task over a transcript, flatten it to tagged text in one user message and explicitly forbid continuing it.

Related: [[transcript-serialization-for-summary]] · [[summarizer-emits-tool-calls]] · [[structured-compaction-summary]]
