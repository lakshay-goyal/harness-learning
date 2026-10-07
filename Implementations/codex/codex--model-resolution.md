---
type: implementation
harness: codex
concept: model-resolution
commit: 622e9e3696
files: [codex-rs/models-manager/src/manager.rs:768-800, codex-rs/models-manager/src/manager.rs:845-851, codex-rs/models-manager/src/model_info.rs:126-132, codex-rs/core/src/compact_model_fallback.rs:10-65]
---
[[model-resolution]] in [[codex]]. Resolution is minimal: exact slugs, no fuzzy or glob syntax.

## Mechanism
- **Default model** (`get_default_model`, `codex-rs/models-manager/src/manager.rs:768-800`):
  - With `allow_provider_model_fallback`: use the requested slug only if it is in the provider's available list, else `default_model_from_available`.
  - Without it: the requested slug as given, else the default from available.
- **`default_model_from_available`** picks the first catalog preset with `is_default`, else the first preset, else an empty string (`codex-rs/models-manager/src/manager.rs:845-851`). The default is therefore **catalog-controlled**: the vendor's remote catalog chooses it ([[codex--model-catalog|codex catalog]]).
- **Unknown slugs are accepted.** They get fallback metadata (272k window, generic `BASE_INSTRUCTIONS`) with the warning "Unknown model {slug} is used. This will use fallback model metadata." (`codex-rs/models-manager/src/model_info.rs:97-132`). Compare pi, which clones the provider default's metadata.
- **No level suffix.** Reasoning effort is a separate setting (`model_reasoning_effort`); there is no `model:high` syntax ([[thinking-level-abstraction]]).
- **Model recorded in history vs model now served.**
  - Pre-sampling compaction deliberately uses the *previous* turn's model when the compaction hash (`comp_hash`) changed or when switching to a smaller window.
  - If the backend rejects that model, for example a retired slug on a resumed ChatGPT thread, `compact_model_fallback` retries once with the currently selected model. This applies to the OpenAI provider with Codex-backend or API-key auth, and not on abort, interrupt or budget errors. It is counted by `codex.compaction.model_fallback` (`codex-rs/core/src/compact_model_fallback.rs:10-65`).

## Evolution
- `172ab264bd` 2026-07-07 (#30319): retry a rejected previous-model compaction with the selected model.
- `92938d880e` 2026-07-13: spawned agents could request models from a different multi-agent backend; fixed ([[model-switch-replays-unsupported-content]]).

## Versus pi
- [[pi--model-resolution|pi]] resolves `provider/id`, bare, fuzzy and glob references with a `:level` suffix, a startup cascade preferring authenticated providers, and scoped cycling.
- Codex has one provider per session and exact slugs; the catalog's `is_default` replaces the startup cascade.

## Failures
- [[compaction-pinned-to-unavailable-model]]
