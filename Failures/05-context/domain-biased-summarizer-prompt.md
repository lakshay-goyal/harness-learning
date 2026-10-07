---
type: failure
concepts: [transcript-serialization-for-summary, structured-compaction-summary]
harnesses: [pi]
---
**Symptom** — Non-coding agents built on the harness (SDK users, the "vacation" demo domain) got summaries framed as coding-assistant work.

**Root cause** — The summarizer system prompt said "a conversation between a user and an AI **coding** assistant" (`3c6c9e52c`, 2025-12-29).

**Fix · [[pi]]** — `72fd91135` 2026-06-07 (#5401) "neutralize compaction summarization prompt" (also `8c6c8a4ef`): "AI coding assistant" → "AI assistant" (`packages/coding-agent/src/core/compaction/utils.ts:161-163`); CHANGELOG "neutral AI assistant wording for non-coding agents".

**Lesson** — A harness-level side prompt must be domain-neutral; domain flavor belongs to the embedding application.

Related: [[transcript-serialization-for-summary]] · [[structured-compaction-summary]] · [[summarizer-refusal]] · [[sdk-embedding]]
