---
type: failure
concepts: [image-normalization, cross-provider-handoff]
harnesses: [pi]
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

**Lesson** — Test image-in-tool-result explicitly per provider and model generation, and gate it on the parsed version.

Related: [[image-normalization]] · [[cross-provider-handoff]] · [[unified-provider-api]] · [[placeholder-text-misleads-model]]
