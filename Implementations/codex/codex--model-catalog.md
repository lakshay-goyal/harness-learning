---
type: implementation
harness: codex
concept: model-catalog
commit: 622e9e3696
files: [codex-rs/models-manager/models.json, codex-rs/models-manager/src/lib.rs:13-15, codex-rs/models-manager/src/manager.rs:34-35, codex-rs/models-manager/src/manager.rs:516-542, codex-rs/models-manager/src/manager.rs:595-635, codex-rs/models-manager/src/manager.rs:653-675, codex-rs/models-manager/src/model_info.rs:16-90, codex-rs/models-manager/src/model_info.rs:126-132, codex-rs/model-provider/src/models_endpoint.rs:44-47, codex-rs/codex-api/src/endpoint/models.rs:34-44, codex-rs/protocol/src/openai_models.rs:391-393, codex-rs/protocol/src/openai_models.rs:411-575, codex-rs/model-provider/src/provider.rs:91-101]
---
[[model-catalog]] in [[codex]]. The catalog is a **vendor-pushed remote control plane**: it carries metadata, prompt text and harness behavior flags.

## Mechanism
- **Layers**:
  1. Bundled `codex-rs/models-manager/models.json`: 11 models at HEAD, 472 KB, compiled in with `include_str!` (`codex-rs/models-manager/src/lib.rs:13-15`).
  2. Remote catalog from the provider's `/models?client_version=X.Y.Z` (`codex-rs/codex-api/src/endpoint/models.rs:34-44`; `codex-rs/model-provider/src/models_endpoint.rs:44-47`).
  3. On-disk cache `models_cache.json` with a 300 s TTL, keyed by client version and auth identity; a mismatch logs "models cache: provider or auth identity mismatch" (`codex-rs/models-manager/src/manager.rs:34-35,760`).
  4. Config overrides (`codex-rs/models-manager/src/model_info.rs:19-55`).
  5. Fallback metadata for unknown slugs, `model_info_from_slug`: warns "Unknown model {slug} is used. This will use fallback model metadata." Defaults are a 272_000 context window, 95% effective, tool-output truncation of 10_000 bytes, and the `BASE_INSTRUCTIONS` prompt (`codex-rs/models-manager/src/model_info.rs:126-132`).
- **Freshness.** Responses carry `x-models-etag`. If it differs from the cached etag the manager refetches; otherwise it only renews the cache TTL (`codex-rs/models-manager/src/manager.rs:516-542`). Rationale in `66b7c673e9`: "so we don't mutate it mid-turn".
  - Fetch timeout is `MODELS_REFRESH_TIMEOUT` = 5 s; the body is capped at `MAX_MODEL_CATALOG_BYTES` = 1 MiB (`codex-rs/model-provider/src/models_endpoint.rs:44-47`).
  - A failed fetch for an explicit provider clears both the in-memory and on-disk models, so the next restart is a forced miss (`codex-rs/models-manager/src/manager.rs:595-635`).
  - API-key users get the bundled catalog only, unless discovery is enabled (`codex-rs/models-manager/src/manager.rs:653-675`).
- **Per-model fields that drive the harness** (`ModelInfo`, `codex-rs/protocol/src/openai_models.rs:411-510`):
  - window and compaction: `context_window`, `max_context_window` (cap for a user override), `auto_compact_token_limit`, `effective_context_window_percent` (default 95, `codex-rs/protocol/src/openai_models.rs:391-393`);
  - `truncation_policy`: bytes or tokens plus a limit. Every bundled model uses tokens/10_000 ([[tool-output-truncation]]);
  - reasoning and output: default and supported reasoning levels, reasoning-summary support and default, verbosity support and default;
  - tool shapes: `shell_type`, `apply_patch_tool_type` (freeform only since `e783341b70`), `web_search_tool_type`, `supports_search_tool`, `experimental_supported_tools`;
  - wire and serving: `input_modalities`, `service_tiers`, `use_responses_lite`, `supports_reasoning_effort_updates`;
  - `comp_hash`: a compaction-compatibility id;
  - instruction-fragment gates: `include_skills_usage_instructions`, `include_plugin_usage_instructions`, `include_apps_usage_instructions`.
  - harness-shape selectors (`codex-rs/protocol/src/openai_models.rs:478-512`): `tool_mode` `direct | code_mode | code_mode_only` overrides the `CodeMode`/`CodeModeOnly` feature flags (`codex-rs/core/src/tools/mod.rs:75-85`) → [[code-mode]]; `multi_agent_version` `disabled | v1 | v2` (`codex-rs/protocol/src/protocol.rs:3141-3145`) + `multi_agent_reasoning_effort` (used when the user picks Ultra); `node_repl_disabled` / `node_repl_auto_review_required`; `supports_experimental_context` gates `ContextManagement` startup (also requires Codex-backend routes + OpenAI auth, no custom key/auth) (`codex-rs/core/src/session/token_budget.rs:21-35`) → [[session-token-budget]]; `model_specialty`.
  - serving/UX metadata: `supported_in_api`, `additional_speed_tiers`, `default_service_tier`, `available_access_programs`, `availability_nux`, `supports_reasoning_summary_parameter` (accepts `reasoning.summary`), `default_reasoning_summary`, `default_verbosity`, `supports_image_detail_original` ([[image-normalization]]); internal `used_fallback_model_metadata` set by core for unknown slugs (`codex-rs/protocol/src/openai_models.rs:420-478`).
- **Model-owned prompt text** `model_messages` (`codex-rs/protocol/src/openai_models.rs:540-575`):
  - `instructions_template` ("Formerly a personality template, now literal text");
  - `content_filter_guidance`, falling back to the bundled text when over 512 UTF-8 bytes;
  - persistent-mode instructions;
  - tool, approval and permission messages;
  - `token_budget` defaults.

  See [[per-model-system-prompt]].
- **Derived limits:**
  - Usable window = `context_window × effective_percent / 100` (`codex-rs/protocol/src/openai_models.rs:521-525`).
  - Auto-compact limit = `min(configured, 90% of context_window)` (`codex-rs/protocol/src/openai_models.rs:527-538`).
- **Overrides** (`codex-rs/models-manager/src/model_info.rs:20-90`):
  - `model_context_window` is clamped to `max_context_window`.
  - `tool_output_token_limit` rewrites the truncation policy and keeps its mode.
  - `base_instructions` replaces the template.
  - `personality = none` strips the `# Personality` H1 section.
- **Bundled catalog at HEAD:**
  - gpt-6-astra, gpt-6.1-sol, gpt-6-sol, gpt-6-luna;
  - gpt-5.6-sol/terra/luna: 272k context, 872k max, Responses Lite;
  - gpt-5.5: 272k/272k, not Lite;
  - hidden: gpt-daybreak-*, codex-auto-review.
- **Side-task model defaults** (`codex-rs/model-provider/src/provider.rs:91-101`):
  - approval review: `codex-auto-review` (API key: `gpt-5.6-luna`);
  - memory extraction: `gpt-5.6-luna`;
  - memory consolidation: `gpt-5.6-terra`.

## Constants
| name | value | path:line |
|---|---|---|
| catalog cache TTL | 300 s | `codex-rs/models-manager/src/manager.rs:35` |
| `MODELS_REFRESH_TIMEOUT` | 5 s | `codex-rs/model-provider/src/models_endpoint.rs:44` |
| `MAX_MODEL_CATALOG_BYTES` | 1 MiB | `codex-rs/model-provider/src/models_endpoint.rs:47` |
| effective context window percent default | 95 | `codex-rs/protocol/src/openai_models.rs:391-393` |
| auto-compact clamp | min(config, 90% window) | `codex-rs/protocol/src/openai_models.rs:527-538` |
| unknown-model fallback | 272_000 window, 95%, 10_000-byte tool output | `codex-rs/models-manager/src/model_info.rs:126-132` |
| content-filter guidance cap | 512 UTF-8 bytes | `codex-rs/protocol/src/openai_models.rs:547-549` |

## Evolution
- `136b3ee5bf` 2025-08-04 (#1838): hardcoded `ModelFamily` abstraction.
- `049a61bcfc` 2025-10-20 (#5292): auto-compact at about 90%: "Users now hit a window exceeded limit and they usually don't know what to do."
- `222a491570` 2025-12-08 (#7722): remote catalog with a disk TTL and etag.
- `b7fa7ca8e9` 2025-12-11: effective context window percent.
- `66b7c673e9` 2026-01-01 (#8491): refresh on etag mismatch.
- `9179c9deac` 2026-01-07 (#8763): `ModelFamily` merged into `ModelInfo` ("Add compaction limit and visible context window to ModelInfo").
- `a1abd53b6a` 2026-02-09 (#11238): the offline fallback was removed. Per-model `include_str!` prompts were deleted, so the catalog is the only source ([[no-offline-per-model-prompts]]).
- `40de788c4d` 2026-02-11: clamp auto-compact to 90% of the window ([[auto-compact-threshold-exceeds-window]]).
- `281b0eae8b` 2026-02-17: setting `model_supports_reasoning_summaries=false` became a no-op ([[thinking-off-not-honored]]).
- `5bb193aa88` 2026-04-17: max context window metadata.
- `0db6811b7c` 2026-04-24: Bedrock models use the function `apply_patch` instead of the bundled freeform type ([[endpoint-rejects-request-field]]). The function variant was deleted entirely in `e783341b70` 2026-05-08.
- `86b1123ff6` 2026-08-14: `supports_parallel_tool_calls` removed from the catalog; parallel calls are always requested.
- `977193486d` 2026-09-16: catalog decode errors stopped echoing the payload ([[error-diagnostics-echo-payload]]).

## Quirks
- The authoritative catalog is per account and per client version, served by the vendor. Prompt text and tool shapes can change without a client release.
- Bedrock has its own catalog (`codex-rs/model-provider/src/amazon_bedrock/catalog.rs`).

## Versus pi
- [[pi--model-catalog|pi]] generates a typed catalog at build time from public aggregators and adds a remote overlay with an ETag. It carries prices and compat flags.
- Codex's catalog carries no prices (cost is server-side, [[usage-cost-accounting]]) but does carry prompts, truncation policy and tool wire types. It is a behavior-configuration channel, not just metadata. See [[single-vs-per-model-system-prompt]].

## Failures
- [[compaction-pinned-to-unavailable-model]]
- [[endpoint-rejects-request-field]]
- [[thinking-off-not-honored]]
- [[error-diagnostics-echo-payload]]
- [[auto-compact-threshold-exceeds-window]]
