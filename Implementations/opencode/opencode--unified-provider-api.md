---
type: implementation
harness: opencode
concept: unified-provider-api
commit: ecc4916b5a
files: [packages/opencode/src/session/llm.ts:85-381, packages/opencode/src/session/llm/native-runtime.ts:23-68, packages/opencode/src/session/llm/ai-sdk.ts:23-120, packages/opencode/src/provider/transform.ts:465-518, packages/opencode/src/effect/runtime-flags.ts:54, packages/llm/src/route/client.ts:279-295, packages/core/src/session/runner/model.ts:142-203, package.json, packages/llm/src/protocols/openai-responses.ts:389-419]
---
[[unified-provider-api]] in [[opencode]].

## Mechanism

### Legacy runtime — Vercel AI SDK behind one `LLMEvent` stream
- `LLM.Service.stream` resolves language model, config, provider, auth; prepares the request; then **per request** either the native `@opencode-ai/llm` route (flag `OPENCODE_EXPERIMENTAL_NATIVE_LLM` and a gate that accepts OpenAI, opencode-managed OpenAI-compatible, Anthropic API-key; not OAuth / missing key) or AI SDK `streamText` whose `fullStream` is adapted to `LLMEvent`s (`packages/opencode/src/session/llm.ts:85-381`; `packages/opencode/src/session/llm/native-runtime.ts:23-68`; `packages/opencode/src/session/llm/ai-sdk.ts`).
- Provider quirks live in **one hotspot**, `ProviderTransform` (`packages/opencode/src/provider/transform.ts`, ~1900 lines), applied as AI SDK `wrapLanguageModel` middleware on the final prompt: `message()` = `unsupportedParts` → `normalizeMessages` (surrogates, Anthropic/Bedrock empty filtering, tool-call id scrub, Mistral "Done." bridge, DeepSeek reasoning padding, interleaved reasoning field) → caching markers → providerOptions key remap → Responses `itemId` strip (`transform.ts:465-518`). Plus `options()`, `variants()`, `schema()`, `temperature/topP/topK`, `maxOutputTokens`.
- Unknown finish → `"unknown"` (loop continues); provider-executed tools carried as `metadata.providerExecuted` and never drive continuation.
- Copilot billing: raw chunks parsed for `total_nano_aiu`, which overrides computed cost (`ai-sdk.ts:33-45`).
- **Vendored SDK patches** (`patches/*`, root `package.json` `patchedDependencies`) — fixes shipped as diffs to `dist/` and `src/` of `@ai-sdk/anthropic` (thinking `blockBinding`), `@ai-sdk/amazon-bedrock`, `@ai-sdk/openai` (service-tier allow-list removed), `@ai-sdk/openai-compatible` (structured stream errors), `@ai-sdk/google` (empty `contents` popped), `@ai-sdk/groq`/`xai`/`mistral` (effort enum → string, Mistral reasoning replay, `prompt_cache_key`), `@modelcontextprotocol/sdk` (session recovery). Three needs: reasoning replay fidelity, SDK enums/allow-lists lagging providers, error/stream fidelity.

### v2 runtime — `packages/llm` (route-first, own protocols)
- Route = Protocol × Endpoint × Auth × Framing (`Route.make`); OpenAI-compatible deployments are "a 5-15 line `Route.make(...)` call" (`packages/llm/AGENTS.md`). `LLM.stream`/`generate` each make **exactly one provider turn**; the package's in-memory tool loop is slated for removal (`specs/v2/todo.md:46-47`).
- Own protocols: Anthropic Messages, OpenAI Responses (HTTP + WebSocket), OpenAI Chat / OpenAI-compatible (9 base-URL profiles), Gemini, Bedrock Converse.
- Runner accepts only three native routes: `@ai-sdk/openai` → Responses, `@ai-sdk/anthropic` → Messages, `@ai-sdk/openai-compatible` with URL → Chat; anything else fails `UnsupportedApiError`, never a silent fallback (`packages/core/src/session/runner/model.ts:142-203`).
- Non-goals: registry, orchestration, persistence, permissions, billing (`packages/llm/DESIGN.md:33-42`).
- **Provider-executed (hosted) tools** pass through untouched: they arrive as `tool-call`/`tool-result` events with `providerExecuted`, are published to history but never dispatched locally (`packages/core/src/session/runner/llm.ts:252`). OpenAI Responses replays hosted output as a stored `item_reference` when `store !== false` and skips it otherwise (`packages/llm/src/protocols/openai-responses.ts:389-419`); stateless (`store: false`) hosted continuation is an open TODO (`specs/v2/todo.md:137`). Hosted results stay outside generic tool-output bounding and need provider-aware pruning because payloads must round-trip exactly (`CONTEXT.md:199`; `specs/v2/session.md:215`).

## Constants
| name | value | path:line |
|---|---|---|
| native runtime flag | `OPENCODE_EXPERIMENTAL_NATIVE_LLM` (`bool`, not umbrella `OPENCODE_EXPERIMENTAL`) | `packages/opencode/src/effect/runtime-flags.ts:54` |
| v2 Anthropic fallback max output | 4096 | `packages/llm/src/protocols/anthropic-messages.ts:510` |
| v2 media caps | 28 MiB encoded / 20 MiB decoded | `packages/llm/src/protocols/shared.ts:162-163` |

## Evolution
- 2025-06-14 `fa1266263d` downgrade to AI SDK v4; 2025-12-14 `fed4776451` "LLM cleanup" (single retry layer, one step per call).
- 2026-01-19 `fc6c9cbbd2` Copilot GPT-5+ routed to Responses; 2026-01-28 `92eb982863` Copilot Anthropic endpoint flip-flop undone.
- 2026-05-08 `5bb7b23440` native LLM core; 2026-05-18 `dbe36851bc` native runtime preview behind a flag; 2026-05-20 `41f6daf96a` route-first API.
- 2026-06-02 `70a2e846cb` Gemini empty replay patched in SDK; 2026-08-05 `709c195905` openai-compatible error chunks preserved by patch; 2026-09-01 `3f39a329c3` Anthropic block-binding patch; 2026-09-06 `ea2d59d7ca` OpenAI service tiers patch.
- v2 rewrite re-hit replay bugs within weeks: `16fb6dac8d` reasoning deltas missing, `eb84f461b8` summary blocks merged, `f254476043` stateless item ids, `11d2f3e5f8` reasoning/text lifecycle (2026-05..06).

## Quirks / drift
- `packages/opencode/src/session/llm/AGENTS.md` says the umbrella `OPENCODE_EXPERIMENTAL=true` enables native, but the flag uses `bool(...)` (`runtime-flags.ts:54`) — doc/code drift.
- A typed SDK in front of fast-moving providers is itself a failure source: ~20 fixes shipped as patches rather than waiting for upstream.

Failures: [[stop-reason-mapping-gaps]] · [[empty-payload-rejections]] · [[endpoint-rejects-request-field]] · [[stream-delta-assembly-errors]] · [[sdk-enum-lags-provider-options]] · [[truncated-stream-accepted-as-success]].

Contrast: [[pi--unified-provider-api|pi]] owns every adapter (pi-ai) from the start; opencode wrapped the AI SDK, patched it, then started its own `packages/llm` for v2.
