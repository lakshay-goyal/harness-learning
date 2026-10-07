---
type: failure
concepts: [unified-provider-api, errors-as-stream-events]
harnesses: [pi, codex]
---
**Symptom**
- Google responses with a tool call but a `MAX_TOKENS` or error stop were treated as normal tool use, hiding truncation (#8059).
- Ollama and LM Studio `finish_reason:"end"` threw an error (#2142).
- The Anthropic `sensitive` stop reason crashed parsing (#978).
- Refusals looked like normal stops and lost `stop_details` (#5666).
- Bedrock reported "An unknown error occurred" for guardrail and content-filter stops (#6598).
- z.ai `finish_reason:"network_error"` was persisted as success (#2313).
- Responses `response.failed` lost its code.

**Root cause** The stop-reason maps were partial: some reasons were missing from SDK types, and some vendors sent non-standard values. Tool presence also overrode the mapped reason unconditionally.

**Fix · [[pi]]**
- `5093641a5` 2026-08-14: promote to `toolUse` only when the mapped reason is `stop` (`packages/ai/src/api/google-generative-ai.ts:227`; `google-vertex.ts:235`) (#8059).
- `a79ca4119` 2026-03-14: map `end` to stop, and unknown values to error "Provider finish_reason: X" (`packages/ai/src/api/openai-completions.ts:1562-1586`) (#2142).
- `d914d1c19` 2026-03-17: z.ai `network_error` becomes an error (#2313). `8e3bb4ff5` 2026-03-17: "provider returned error" made retryable (#2264).
- `ee7c0a7d1` 2026-01-28: `sensitive` becomes error "Provider stopped with: sensitive" (`packages/ai/src/api/anthropic-messages.ts:1613-1639`) (#978).
- `a455f62f7` 2026-06-12: refusal handling (#5666). `eb1f87fa9` 2026-08-17: `refusal` becomes error carrying `stop_details.explanation`, or "The model refused to complete the request".
- `f8f75544b` 2026-07-13: Bedrock guardrail and other reasons become error "Provider stopped with: <reason>". `model_context_window_exceeded` becomes `length` (`packages/ai/src/api/bedrock-converse-stream.ts:1177-1192`) (#6598).
- `d9cfa115b` 2026-03-08: Responses `response.failed` throws with code/message or the incomplete reason (`openai-responses-shared.ts:747-757`).
- Raw reason preserved in `rawStopReason`: `926eb15c1` (Anthropic), `637737ca7` (Bedrock), `fe1c9b6d5` (completions) and `23cb385b6` (Google), all 2026-07-29. `c3e7bc60a` 2026-08-07: Codex `end_turn` recorded to `output.endTurn`.
- Current exhaustive `never` checks: Google `google-shared.ts:441-469`; Responses status map `openai-responses-shared.ts:779-809`.

**Fix · [[codex]]** — Chat Completions era, through proxies.
- Symptom: Claude models behind OpenAI-compatible proxies returned only 1–4 characters. The proxies sent `finish_reason: ""` mid-stream, and Codex treated any non-null `finish_reason` as termination (`de1768d3ba` 2025-11-17, `de1768d3ba:codex-rs/core/src/chat_completions.rs`).
- Related fixes:
  - `bba5e5e0d4` 2026-01-05: handle the Chat Completions `[DONE]` sentinel.
  - `acb8ed493f` 2025-12-08: a missing tool name crashed LiteLLM.
  - `649badd102` / `5f80ad6da8`: multiple and parallel tool calls were broken on chat.
- Resolution: the whole Chat Completions path was deleted in `d2394a2494` 2026-02-03 ([[no-chat-completions-wire]]).
- What remains on Responses:
  - `response.incomplete` with reason `content_filter` → ContentFilter.
  - Any other reason except `interrupted` → a retryable Stream error (`codex-rs/codex-api/src/sse/responses.rs:418-433`).

**Lesson** — Make stop-reason maps total, with an explicit "unknown means error, keep the raw reason" arm, and never let tool presence override a length or error stop. Compatibility layers for a second wire protocol accrue a long tail of such quirks; deleting the layer is a legitimate fix.

Related: [[unified-provider-api]] · [[server-side-refusal-fallback]] · [[terminal-event-required]] · [[truncated-stream-accepted-as-success]] · [[pi--unified-provider-api|pi]] · [[codex--unified-provider-api|codex]] · [[no-chat-completions-wire]] · [[provider-breadth]]
