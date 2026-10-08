---
type: failure
concepts: [thinking-level-abstraction, model-catalog]
harnesses: [pi, codex]
---
**Symptom** — User picked "off" but the model still thought, or the request 400'd:
- Gemini dynamic-thinking default kept thinking when `reasoning` was undefined.
- z.ai got the wrong param name and burned thinking tokens.
- Anthropic: no `thinking` field meant the model default, which thinks.
- Claude Fable 5 rejected `{type:"disabled"}`.
- Inception Mercury 2 with `effort:"none"` silently disabled tool calling.
- Copilot gpt-5-mini returned 400 on `effort:"none"` during summaries.
- Codex Off was omitted, so the backend default (on) applied.
- Gemini 3 Pro cannot disable thinking at all, and Gemini 3 Flash/Flash-Lite cannot fully disable it.

**Root cause** — "off" has three wire encodings: omit the field, send an explicit disable (`thinking:{type:"disabled"}`, `enable_thinking:false`, `thinkingBudget:0`), or send an effort of `"none"`. Each model accepts a different subset, some reject the other encodings, and some change capability (tools) when switched off. The adapters assumed "omit = off".

**Fix · [[pi]]**
- `fbda78bfb` 2025-12-15 — explicitly disable when `reasoning` is undefined (`packages/ai/src/stream.ts:136,168` at `fbda78bfb`; file removed in `8a0903ebf`) (#180). HEAD Google: streamSimple with no `reasoning` → `thinking:{enabled:false}` (`packages/ai/src/api/google-generative-ai.ts:319-321`).
- `22b3be834` 2026-02-28 — `enable_thinking` for z.ai instead of the `thinking` param (#1674).
- `6129971c0` 2026-03-22 — Anthropic sends `{type:"disabled"}` when off (#2022). HEAD `packages/ai/src/api/anthropic-messages.ts:1261-1263`.
- `d1613e3f5` 2026-03-22 — explicit off across providers (#2490, release 0.62.0). Google level models fall back to the lowest supported level without `includeThoughts`, so hidden thinking is not surfaced (`packages/ai/src/api/google-shared.ts:102-111`).
- `bab58f821` 2026-03-24 — omit the Copilot Responses reasoning default (#2567) (`packages/ai/src/api/openai-responses.ts:374-378`).
- `e2b69a0bb` 2026-05-13 — Mercury 2 `thinkingLevelMap.off = null` in the generator (`packages/ai/scripts/generate-models.ts:243` at that commit; HEAD `:1157`).
- `9ccfcd7cf` 2026-06-10 — omit disabled thinking for Fable 5. Disable is gated on `thinkingLevelMap.off !== null` (`anthropic-messages.ts:1261-1263`; tests `packages/ai/test/anthropic-thinking-disable.test.ts:113-155`).
- `e86102f18` 2026-09-10 — Codex receives `{effort: off ?? "none"}` explicitly (#9191) (`packages/ai/src/api/openai-codex-responses.ts:582-597`).
- HEAD per-format off encodings: `packages/ai/src/api/openai-completions.ts:880-977`. Managed-effort Anthropic models force `off:null` (`generate-models.ts:831`).

**Fix · [[codex]]** — the inverse: a config knob disabled reasoning.
- Symptom: `model_supports_reasoning_summaries` was meant only to *enable* reasoning for unknown custom models, but it could disable reasoning for known reasoning models such as `gpt-5.3-codex`. It was "difficult to diagnose… led to many support issues" (`281b0eae8b` body).
- `281b0eae8b` 2026-02-17: setting it to false is a no-op (`281b0eae8b:codex-rs/core/src/models_manager/model_info.rs`).
- Related:
  - `0af7e4a195` 2025-12-11: omit the summary when it is None.
  - `dffe1f02a3` 2026-07-10: respect model support for reasoning summaries.
  - `d2d00b6632` 2026-07-10: always send reasoning parameters.
  - `3e4707b34b` 2026-08-26: `persistent` effort is sent as an explicit `disabled`.

**Lesson** — Model "off" as a per-model wire value, with `null` meaning "cannot be disabled"; never assume that omitting the field means off. Capability overrides should only add a capability, unless the knob is explicitly named as a disable.

Related: [[thinking-level-abstraction]] · [[pi--thinking-level-abstraction|pi]] · [[model-catalog]] · [[thinking-config-per-model-drift]] · [[codex--thinking-level-abstraction|codex]]
