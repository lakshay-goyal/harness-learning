---
type: implementation
harness: opencode
concept: model-resolution
commit: ecc4916b5a
files: [packages/opencode/src/provider/provider.ts:1894-1916, packages/opencode/src/provider/provider.ts:1961-2086, packages/opencode/src/session/prompt.ts:646, packages/core/src/session/runner/model.ts:142-203]
---
[[model-resolution]] in [[opencode]].

## Mechanism

### Legacy runtime
- Model chosen **per user message**: `input.model ?? agent.model ?? session's current model` (`packages/opencode/src/session/prompt.ts:646`). References are `provider/model` (`parseModel`); unknown ids → `ModelNotFoundError` with fuzzy `suggestions` (`packages/opencode/src/provider/provider.ts:1894-1916`).
- **Small model** (`getSmallModel`, for side calls): config `small_model` → plugin hook `experimental.provider.small_model` → family priority; opencode provider `["gpt-nano"]`; Copilot `gpt-mini` first; Bedrock prefers `global.`/regional prefixes; Azure none (`provider.ts:1961-2028`).
- **Default model**: config `model` → most recent valid entry in state `model.json` → first configured provider's top model by `priority` (`provider.ts:2030-2086`).

### v2 runtime
- `SessionRunnerModel.resolve`: explicit session model never silently falls back; otherwise catalog default if supported, else first available supported route; only three native routes accepted, else `UnsupportedApiError` / `ModelNotSelectedError` (`packages/core/src/session/runner/model.ts:142-203`). "`openai/responses` with WebSocket transport must not silently downgrade to HTTP" (`specs/v2/provider-model.md:284`).

## Constants
| name | value | path:line |
|---|---|---|
| default model priority | `gpt-5`, `claude-sonnet-4`, `big-pickle`, `gemini-3-pro` | `packages/opencode/src/provider/provider.ts:2069` |
| small model family priority | `gemini-flash`, `gpt-nano`, `claude-haiku` | `packages/opencode/src/provider/provider.ts:2070` |

## Evolution
- 2026-01-12 `71a7ad1a4e` title generation uses the user's model, not the assistant's.
- 2026-06-25 `ded29f03f0` refined small model defaults.

## Quirks / drift
- Per-message model choice makes mid-session switches routine, which is why [[opencode--cross-provider-handoff|cross-provider-handoff]] matters.

Failures: [[unusable-default-model-selected]] (v2 `supported()` filter is the guard; no opencode fix commit).

Contrast: [[pi--model-resolution|pi]] resolves fuzzy/glob/`:level` references with a startup cascade; opencode resolves `provider/model` per message plus a separate small-model cascade for side calls.
