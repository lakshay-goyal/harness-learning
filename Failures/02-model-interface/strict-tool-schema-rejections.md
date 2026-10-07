---
type: failure
concepts: [constrained-tool-sampling, tool-schema-normalization, mcp-integration]
harnesses: [pi, codex]
---
**Symptom** — Provider 400s on tool declarations:
- `strict` was sent to OpenAI-compatible servers that don't support it.
- LM Studio wanted a boolean `strict`.
- Cerebras rejected mixed strict and non-strict tools.
- Anthropic strict mode rejected `minimum/maximum`, so one bounded-integer tool broke **every** turn.
- Cloud Code Assist rejected `$schema/$defs/definitions`.
- Mistral SDK validation choked on TypeBox symbol keys.
- Google rejected anyOf/oneOf in OpenAPI `parameters`.

**Root cause** — Each provider accepts a different JSON-Schema subset and a different strict-mode contract. pi sent one schema shape everywhere and defaulted strict on for unknown endpoints.

**Fix · [[pi]]**
- 0.42.2 — boolean strict for LM Studio (#598).
- 0.51.0 — omit `strict` when unsupported (#1172).
- `1caadb2e2` 2026-02-08 — Google uses `parametersJsonSchema` (full JSON Schema) (#1398).
- `2dddc5ba2` 2026-04-18 — `stripSymbolKeys` for Mistral (#3361) (`packages/ai/src/api/mistral-conversations.ts:763-792`).
- `f732f5e85` 2026-04-19 — `sanitizeForOpenApi` strips `$schema,$id,$anchor,$dynamicAnchor,$vocabulary,$comment,$defs,definitions` (#3412) (`packages/ai/src/api/google-shared.ts:345-370`). This path is now dead after fe66edd94.
- `7915cdac6` 2026-08-11 — `makeStrictJsonSchema` conversion (`packages/ai/src/api/constrained-sampling.ts:56-140`).
- `fcff255b0` 2026-09-05 — built-in tools default to `strict:"prefer"`, which raised exposure to rejections (`packages/coding-agent/src/core/tools/read.ts:101`, `bash.ts:260`, `edit.ts:156`, `write.ts:57`).
- `af7359b90` 2026-09-20 — exclude Cerebras from `supportsStrictMode` (#9804).
- `890f92088` 2026-09-21 — the runtime default for unknown OpenAI-compatible providers is non-strict, and the catalog opts verified models in (#9816) (`packages/ai/src/api/openai-completions.ts:1674-1675`; generator `packages/ai/scripts/generate-models.ts:781`).
- `295cc72b0` 2026-09-30 — per-provider keyword veto (`packages/ai/src/api/anthropic-messages.ts:1541-1574`) (#9953):
  - `prefer` falls back silently to non-strict, and `require` throws (`constrained-sampling.ts:225-249`).

**Fix · [[codex]]** — the schema sanitizer invented constraints or kept them.
- Untyped nodes: MCP and dynamic tool schema nodes without `type` were filled in as `string` by `sanitize_json_schema`. This invented a scalar constraint that "could incorrectly steer tool arguments away from the provider's actual accepted shape". `4dbca61e20` 2026-05-18 defaults them to `{}` (`codex-rs/tools/src/json_schema.rs:71-206`; [[invented-schema-constraint]]).
- Bounds: `minimum`/`maximum`/`maxLength` were kept in generated declarations. `ac644ed112` 2026-08-26 stopped preserving them.
- Oversized schemas: client tool schemas over 4k bytes are compacted, which stripped field guidance from the web-search tool. `9fe55d68e6` 2026-05-26 added a bypass.

**Lesson** — Constrained sampling needs per-provider schema-subset validation with a graceful "prefer" fallback, and unknown endpoints should default to the lenient wire subset. Schema normalization must not change meaning: never invent types, and never drop field guidance.

Related: [[constrained-tool-sampling]] · [[pi--constrained-tool-sampling|pi]] · [[model-catalog]] · [[tool-schema-normalization]] · [[codex--constrained-tool-sampling|codex]]
