---
type: failure
concepts: [tool-call-id-normalization]
harnesses: [pi]
---
**Symptom** — Chat Completions failed with "duplicate tool_call_id" after switching from a Responses model (#6854). Gemini returned missing or duplicate tool-call ids, which broke the pairing between calls and results.

**Root cause** — Two sources of collisions:
- Lossy normalization was not injective. Parallel Responses calls can share a `call_id` and differ only by `item_id`, so keeping only the part before `|` produced duplicates.
- Provider-issued ids were assumed to be unique and present.

**Fix · [[pi]]**
- `d9f7f8147` (2026-07-20), #6854: Completions builds `callId_itemId` after sanitizing, and if that is over 40 chars uses `callIdPrefix_<8-char shortHash>`. Plain ids are truncated to 40 only for provider `openai` (`packages/ai/src/api/openai-completions.ts:1202-1226`; CHANGELOG `:546`).
- `6c3580828` (2025-09-04): Google synthesizes `${name}_${Date.now()}_${++toolCallCounter}` when an id is missing or duplicated within a message, using a module-level counter (`packages/ai/src/api/google-generative-ai.ts:56-57,195-201`; vertex `:65-66,204-209`).
- `eb9f1183a` (2026-03-03): Mistral requires exactly 9 alphanumeric chars (`MISTRAL_TOOL_CALL_ID_LENGTH = 9`, `packages/ai/src/api/mistral-conversations.ts:28`). `createMistralToolCallIdNormalizer` hashes to 9 alnum chars, keeps bidirectional maps, and reseeds with `:attempt` on collision (`:237-257`). Streamed missing or `"null"` ids get `deriveMistralToolCallId("toolcall:<index>")` (`:702-705`).

**Lesson** — Id compression must preserve uniqueness, so hash rather than truncate. Never trust provider ids to be unique; dedupe within a message.

Related: [[tool-call-id-normalization]] · [[cross-provider-tool-call-id-normalization]] · [[streamed-tool-call-fragmentation]] · [[pi--tool-call-id-normalization|pi]]
