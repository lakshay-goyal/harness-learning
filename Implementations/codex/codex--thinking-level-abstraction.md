---
type: implementation
harness: codex
concept: thinking-level-abstraction
commit: 622e9e3696
files: [codex-rs/protocol/src/openai_models.rs:60-73, codex-rs/protocol/src/config_types.rs:65-72, codex-rs/protocol/src/config_types.rs:92-97, codex-rs/core/src/client.rs:869-889, codex-rs/core/src/client.rs:954-972, codex-rs/models-manager/models.json, codex-rs/core/src/session/reasoning_effort.rs:136-156]
---
[[thinking-level-abstraction]] in [[codex]]. The vocabulary is the vendor's own: OpenAI `reasoning.effort` values, not a provider-neutral scale.

## Mechanism
- **Effort enum** `ReasoningEffort` (`codex-rs/protocol/src/openai_models.rs:60-73`): `none`, `minimal`, `low`, `medium` (default), `high`, `xhigh`, `max`, `ultra`, `persistent`, and `Custom(String)` for model-defined values the client does not know.
  - `persistent` is sent as `disabled` (`3e4707b34b`). It is the [[persistent-agent-mode]] effort.
  - `ultra` resolution depends on the model (`7f135e1314`).
  - Numeric custom efforts are serialized as JSON numbers (`ae132dc50a`).
- **Per-model levels come from the catalog**: `supported_reasoning_levels`, `default_reasoning_level` (`codex-rs/models-manager/models.json`).
  - gpt-6 and gpt-5.6 models support low..max, plus ultra.
  - gpt-5.5 supports low..xhigh.
  - The value is resolved per model by `model_info.resolve_reasoning_effort` (`codex-rs/core/src/session/reasoning_effort.rs:148-150`).
- **Request** (`build_reasoning`, `codex-rs/core/src/client.rs:869-889`):
  - `reasoning.effort` is the selected effort, or the model default.
  - `reasoning.summary` is sent only if the model supports the summary parameter and the summary is not `none`. The enum is `auto|concise|detailed|none` (`codex-rs/protocol/src/config_types.rs:65-72`).
  - `reasoning.context = all_turns` is set only for Responses Lite models.
  - With concurrent summaries enabled, OpenAI requests get `stream_options.reasoning_summary_delivery = SequentialCutoff` (`codex-rs/core/src/client.rs:954-960`).
- **Verbosity** `low|medium|high` (`codex-rs/protocol/src/config_types.rs:92-97`) is sent in `text.verbosity` only when `model_info.support_verbosity` is set. Otherwise the request logs "model_verbosity is set but ignored as the model does not support verbosity" and drops it (`codex-rs/core/src/client.rs:962-972`).
- **No answer reserve.** Codex sends no `max_output_tokens` at all ([[max-tokens-context-clamp]] absent), so reasoning and the answer are not budget-split client-side.
- **Mid-session changes** are pinned per window and appended as configuration items ([[cache-preserving-config-update]]).

## Evolution
- `80b00a193e` 2025-08-22: verbosity for GPT-5.
- `0af7e4a195` 2025-12-11: omit the summary when it is None.
- `281b0eae8b` 2026-02-17: `model_supports_reasoning_summaries=false` became a no-op; it had disabled reasoning on known reasoning models ([[thinking-off-not-honored]]).
- `8ac304c299` 2026-06-04: model-defined efforts.
- `80f54d1266` 2026-06-29: `max` made first-class.
- `775ef7dcc7` 2026-07-06: sequential-cutoff summary delivery.
- `d2d00b6632` 2026-07-10: always send reasoning parameters.
- `dffe1f02a3` 2026-07-10: respect model support for reasoning summaries.
- `3e4707b34b` 2026-08-26: `persistent` → `disabled`.
- `7f135e1314` 2026-08-27: model-aware `ultra`.
- `ae132dc50a` 2026-09-23: numeric custom efforts.

## Versus pi
- [[pi--thinking-level-abstraction|pi]] maps a neutral scale onto about 11 wire formats, with budgets and an answer reserve.
- Codex needs no mapping layer; its complexity goes into the catalog's per-model supported levels and into vendor-specific modes (`ultra`, `persistent`, `Custom`) the client may not know.

## Failures
- [[thinking-off-not-honored]]
- [[compaction-request-shape-mismatch]]
