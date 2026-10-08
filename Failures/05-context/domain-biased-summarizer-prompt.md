---
type: failure
concepts: [transcript-serialization-for-summary, structured-compaction-summary, auto-compaction]
harnesses: [pi, codex]
---
**Symptom** — Non-coding agents built on the harness (SDK users, the "vacation" demo domain) got summaries framed as coding-assistant work.

**Root cause** — The summarizer system prompt said "a conversation between a user and an AI **coding** assistant" (`3c6c9e52c`, 2025-12-29).

**Fix · [[pi]]** — `72fd91135` 2026-06-07 (#5401) "neutralize compaction summarization prompt" (also `8c6c8a4ef`): "AI coding assistant" → "AI assistant" (`packages/coding-agent/src/core/compaction/utils.ts:161-163`); CHANGELOG "neutral AI assistant wording for non-coding agents".

**Fix · [[codex]]**
- Symptom variant: the first auto-compaction prompt (`ea225df22e` 2025-09-12) was coding-specific and in-character: "You have exceeded the maximum number of tokens, please stop coding and instead write a short memento message for the next agent" with bullets "List outstanding TODOs with file paths / line numbers", "Flag code that needs more tests".
- `611e00c862` 2025-10-31 "feat: compactor 2 (#6027)": task-neutral "You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task." (`codex-rs/prompts/templates/compact/prompt.md:1-9`). A side-branch attempt to bring the memento back (`e67726a994` 2025-11-19) never merged.

**Lesson** — A harness-level side prompt must be domain-neutral; domain flavor belongs to the embedding application, and task specifics are better carried by preserved user messages than by the summarizer's template.

Related: [[transcript-serialization-for-summary]] · [[structured-compaction-summary]] · [[summarizer-refusal]] · [[sdk-embedding]] · [[auto-compaction]] · [[codex--auto-compaction|codex]]
