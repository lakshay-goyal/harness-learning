---
type: implementation
harness: opencode
concept: sampling-parameter-defaults
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:520-574, packages/opencode/src/session/llm/request.ts:114-130, packages/opencode/src/agent/agent.ts:240]
---
[[sampling-parameter-defaults]] in [[opencode]].

## Mechanism (legacy runtime)
- `ProviderTransform.temperature/topP/topK(model)` keyed on lower-cased `model.api.id` substrings; `undefined` = omit, provider default applies (`packages/opencode/src/provider/transform.ts:520-574`).
- Temperature sent only if the catalog says `capabilities.temperature`; agent `temperature`/`topP` override; plugins can rewrite via `chat.params` (`packages/opencode/src/session/llm/request.ts:114-130`).
- Table:

| model family | temperature | topP | topK |
|---|---|---|---|
| Claude | omitted | — | — |
| Gemini 2.5 / 3 flash+pro / 3.1 / 3.5-flash | 1.0 | 0.95 | 64 |
| older Gemini | omitted | omitted | omitted |
| GLM-4.6, GLM-4.7 | 1.0 | — | — |
| MiniMax-M2 | 1.0 | 0.95 | 20 (40 for m2.x/m25/m21) |
| Kimi K2 | 0.6 (thinking / k2.5 → 1.0) | 0.95 for k2.5 | — |
| DeepSeek V4 flash (deepseek / opencode providers) | — | 0.95 | — |
| north-mini-code | 1.0 | — | — |
| internal title agent | 0.5 | — | — |

## Constants
| name | value | path:line |
|---|---|---|
| `GEMINI_MODELS_WITH_SAMPLING_DEFAULTS` | 4 regexes | `packages/opencode/src/provider/transform.ts:523-528` |
| title agent temperature | 0.5 | `packages/opencode/src/agent/agent.ts:240` |

## Evolution
- Not traced commit by commit (table grows per model launch).
- v2: topK forwarded to Bedrock Converse via `additionalModelRequestFields` (`2d993cd0d5` 2026-06-21); no family table in `packages/llm` found (unverified).

## Quirks / drift
- Encodes vendor-recommended sampling in harness code; a renamed model id silently loses its defaults.

Contrast: pi sends no harness-owned sampling defaults (no equivalent found in pi notes).
