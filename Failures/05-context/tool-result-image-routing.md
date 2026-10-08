---
type: failure
concepts: [image-normalization, cross-provider-handoff]
harnesses: [pi, opencode]
---
**Symptom**
- Gemini gave flaky or broken answers when `read` returned images.
- OpenAI Responses silently dropped images in `function_call_output`.
- Images rerouted as follow-up user messages lost their association with the tool call.

**Root cause** — Image-in-tool-result has a different wire shape per provider and model generation, and it was detected by string `includes("gemini-3")` instead of parsing the version.

**Fix · [[pi]]**
- `bf51dd412` 2025-12-21 — Gemini 3: nest images in `functionResponse.parts`; older models get a separate user turn `"Tool result image:"` + inlineData (`packages/ai/src/api/google-shared.ts:315,332-338`).
- `933348593` 2026-03-14 — Responses puts images INSIDE `function_call_output` as an `input_image` list when the model supports images (#2104) (`packages/ai/src/api/openai-responses-shared.ts:81-108,332-349`).
- `453b22397` 2026-03-18 — `supportsMultimodalFunctionResponse` by parsed major version (`/^gemini(?:-live)?-(\d+)/`) for Gemini ≥3 and non-Gemini ids (#2052) (`google-shared.ts:174-186`).
- 0.50.0 — images batched after consecutive tool results (#902). HEAD Chat Completions: images go in ONE synthetic user message "Attached image(s) from tool result:" after the batched results (`packages/ai/src/api/openai-completions.ts:1434-1468`).
- Images are dropped if the model lacks `"image"` input (`google-shared.ts:288-290`). Shared placeholders for non-vision models: `packages/ai/src/api/transform-messages.ts:12-57`.

**Fix · [[opencode]]**
- `de2de099b4` 2026-01-16 stop hoisting tool images into user messages → reverted the same day `f5a6a4af7f`; redone `c2844697f3` 2026-01-21 "ensure images are properly returned as tool results".
- Per-SDK allow-list `supportsMediaInToolResult`; everything else hoisted into a synthetic user message "Attached media from tool result:" (`packages/opencode/src/session/message-v2.ts:46`, `:147-163`). Growth: `72de9fe7a6` 2026-02-05 Kimi via OpenAI-compatible; `c10134729d` 2026-09-20 Bedrock inline only for Claude/Nova/Llama 4; `83802d800e` 2026-10-06 xAI SDK bump so tool-result images reach xAI; xAI non-png/jpeg/webp dropped (`:165-169`).
- `8b9b9ad31e` 2026-04-12 hoisted image messages were billed by GitHub Copilot as user-initiated premium requests.
- v2 native protocols emit structured image blocks inside tool results: `9db90a0b76` (Anthropic), `700d012025` (OpenAI Responses), both 2026-05-22.

**Lesson** — Test image-in-tool-result explicitly per provider and model generation, and gate it on the parsed version.

Related: [[image-normalization]] · [[cross-provider-handoff]] · [[unified-provider-api]] · [[placeholder-text-misleads-model]]
