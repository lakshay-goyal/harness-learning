---
type: implementation
harness: pi
concept: model-resolution
commit: b30a6dd77
files: [packages/coding-agent/src/core/model-resolver.ts:20, packages/coding-agent/src/core/model-resolver.ts:204, packages/coding-agent/src/core/model-resolver.ts:406, packages/coding-agent/src/core/model-resolver.ts:614, packages/coding-agent/src/core/agent-session.ts:2518, packages/coding-agent/src/main.ts:828]
---
[[model-resolution]] in [[pi]].

## Mechanism
- **Default model per provider** table (`packages/coding-agent/src/core/model-resolver.ts:20-62`): anthropic `claude-opus-4-8`, openai `gpt-5.5`, openai-codex `gpt-6.1-sol`, radius `balanced`, github-copilot `gpt-5.4`, openrouter `moonshotai/kimi-k2.6`, … Edited whenever upstream removes models (`49b9df489`, `28fd59086`, `e1787702d`, `6f7551516`).
- **Alias heuristic**: id ending `-latest` or lacking `-YYYYMMDD` suffix = alias (`:74-81`).
- **Exact reference match** `findExactModelReferenceMatch` (`:88-130`): case-insensitive canonical `provider/id` → split at first `/` → bare id; ambiguous → `undefined`.
- **Fuzzy** `tryMatchModel` (`:136-166`): substring on id or name; prefer aliases (sorted desc by id), else latest dated version.
- **`:thinking` suffix** `parseModelPattern` (`:204-257`): try full pattern first (OpenRouter ids like `x:exacto` survive — `9a7863fc9`), else split on **last** colon; valid level → recurse on prefix; invalid suffix: scope mode warns + default, strict CLI mode (`allowInvalidThinkingLevelFallback:false`) fails "to avoid accidentally resolving to a different model" (`:238-244`). Level clamping → [[pi--thinking-level-abstraction|thinking-level-abstraction]].
- **Scoped models** (`--models`, `enabledModels` setting): globs (`*?[`) via `minimatch` against `provider/id` OR id, nocase, optional `:level` (`:291-337`); exact match tried first so bracketed ids are literals (`:306-312`; `da8dd8726`); dedupe via `modelsAreEqual` (`:332, 356`); diagnostics `no-match` / `invalid-thinking-level` (`:270-275`). Resolved only against **available** (authed) models (`:364-370`).
- **CLI `--model`** `resolveCliModel` (`:406-606`) uses **all** models (not only authed) "so --api-key can be used for first-time setup" (`:418-420`). Order:
  1. `--provider` canonicalised case-insensitively (`:430-442`).
  2. Infer provider from prefix before first `/` when it is a known provider — beats gateway ids containing slashes like `zai/glm-5` (`:444-463`; `7364696ae`).
  3. No provider: exact id/canonical matches; multiple → choose sole **authenticated** provider; else error listing candidates + auth hint (`:470-504`; `b04faa2da` #7327).
  4. Strip duplicated provider prefix when both flags given (`:506-512`).
  5. `parseModelPattern` strict within provider candidates (`:514-517`).
  6. Inferred provider unauthenticated but exactly one authenticated provider has the raw full id (e.g. `xiaomi/mimo-v2.5-pro` on commandcode) → take it (`:519-540`; `1b2c32c65` #5643).
  7. Inferred provider without match → full input as raw id across all models (`:548-568`).
  8. **Custom-id fallback**: unknown id on known provider → clone provider default (or first) model with `id=name=<pattern>` (`buildFallbackModel`, `:175-189`), `:thinking` parsed first (`aa0398216`), `reasoning:true` forced when thinking requested (`:589-591`; `1fc80f4f6`), warning "Using custom model id" (`:570-596`; `6f4bd814b` #1759).
- **Initial model cascade** `findInitialModel` (`:614-709`): (1) CLI provider+model (exit on error); (2) first scoped model unless continuing/resuming, thinking = pattern level ?? per-model setting ?? default; (3) saved default **only if authed** (`:675-688`; `ca09b2b1a` #6231); (4) first available matching `defaultModelPerProvider` in table order, else first available; (5) none.
- **Session restore** `restoreModelFromSession` (`:714-783`): model must exist AND be authed; else current model → default-per-provider → first available, with `fallbackMessage`.
- `--api-key` → `modelRuntime.setRuntimeApiKey(selectedModel.provider, key)`; error if no model specified (`packages/coding-agent/src/main.ts:828-837`). Runtime key is non-persistent ([[pi--credential-resolution|credential-resolution]]).
- **Cycling (Ctrl+P)**: scoped list filtered to available, else whole available snapshot; wraps modulo; appends `model_change` session entry; thinking clamped per model; optional persist to default + add to `enabledModels` (`packages/coding-agent/src/core/agent-session.ts:2518-2599, 2498-2510`). `setModel` requires `checkAuth` else throws `No API key for` (`:2476-2479`). Model/thinking changes session-scoped unless persisted (`2ff8ba622` #8356). Extension event `model_select` on set/cycle/restore (`packages/coding-agent/src/core/extensions/types.ts:1104-1109`).
- **Branch model selection**: last `model_change` wins only if virtual; otherwise latest physical response wins; one catalog lookup (`packages/coding-agent/src/core/virtual-models.ts:120-145`; `a0660b174` #10198). See [[pi--virtual-model-router|virtual-model-router]].
- `DEFAULT_THINKING_LEVEL = "medium"` (`packages/coding-agent/src/core/defaults.ts:3-12`).
- RPC surface: `set_model`, `cycle_model`, `get_available_models`, `set/cycle_thinking_level`, `get_available_thinking_levels` (`packages/coding-agent/src/modes/rpc/rpc-types.ts:22-74`).

## Constants
| name | value | path:line |
|---|---|---|
| Default thinking level | medium | packages/coding-agent/src/core/defaults.ts:3-12 |
| Default model per provider | table (anthropic claude-opus-4-8, openai gpt-5.5, …) | packages/coding-agent/src/core/model-resolver.ts:20-62 |
| Glob metacharacters | `*?[` (minimatch, nocase) | packages/coding-agent/src/core/model-resolver.ts:291-337 |
| Alias rule | `-latest` or no `-YYYYMMDD` | packages/coding-agent/src/core/model-resolver.ts:74-81 |

## Evolution
- 2025-12-19 `9a7863fc9` colons in OpenRouter ids (#242): literal match before `:level` split.
- 2026-02-22 `7364696ae` provider/model split beats gateway ids with slashes.
- 2026-03-04 `6f4bd814b` provider-scoped custom model ids (#1759).
- 2026-06-10 `aa0398216` parse `:thinking` in custom-id fallback; 2026-06-12 `1fc80f4f6` keep fallback thinking (force reasoning); `1b2c32c65` authenticated slash-id fallback (#5643).
- 2026-07-02 `ca09b2b1a` skip unauthenticated saved default (#6231).
- 2026-07-23 `da8dd8726` bracketed scoped ids literal.
- 2026-08-03 `b04faa2da` `--model` prefers sole authed provider, else ambiguity error (#7327/#7366).
- 2026-08-19 `2ff8ba622` model/thinking changes session-scoped (#8356).
- 2026-09-30 `a0660b174` branch model selection with one catalog lookup (#10198).
- 2026-09-28..10-03 default table edits `6f7551516`, `e1787702d`, `28fd59086`, `49b9df489`.

## Evidence commits
9a7863fc9, 7364696ae, 6f4bd814b, aa0398216, 1fc80f4f6, 1b2c32c65, ca09b2b1a, da8dd8726, b04faa2da, 2ff8ba622, a0660b174, 49b9df489, 28fd59086, e1787702d, 6f7551516

## Quirks
- CLI resolution sees unauthenticated models (first-time setup) while scoped resolution sees only authed ones — two different universes for the same string.
- `_getThinkingLevelForModelSwitch` per-model default/clamp behaviour not traced in depth (unverified).
- Custom-id fallback silently fabricates metadata (context window, cost) from a sibling model — warning only.

## Failures
- [[model-reference-ambiguity]]
- [[unusable-default-model-selected]]
- [[catalog-hot-path-quadratic]]
