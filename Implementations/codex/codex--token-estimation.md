---
type: implementation
harness: codex
concept: token-estimation
commit: 622e9e3696
files: [codex-rs/core/src/context_manager/history.rs:934, codex-rs/core/src/context_manager/history.rs:893, codex-rs/core/src/context_manager/history.rs:1053, codex-rs/core/src/context_manager/history.rs:1065, codex-rs/core/src/context_manager/history.rs:1074, codex-rs/core/src/context_manager/history.rs:649, codex-rs/core/src/context_manager/history.rs:494, codex-rs/utils/string/src/truncate.rs:4, codex-rs/protocol/src/openai_models.rs:391, codex-rs/protocol/src/openai_models.rs:527, codex-rs/core/src/session/context_window.rs:67]
---
[[token-estimation]] in [[codex]].

## Mechanism
- **Live accounting** = last provider-reported `last_token_usage.total_tokens` + byte-heuristic estimate of every item recorded after the last model-generated item (`get_total_token_usage`, `codex-rs/core/src/context_manager/history.rs:934-951`).
- **Reasoning**: unless the server signals `ServerReasoningIncluded`, the client ADDS estimated tokens of encrypted reasoning items preceding the last user-turn boundary (`get_non_last_reasoning_items_tokens`, `:893-918`); doc: "When true, the server already accounted for past reasoning tokens and the client should not re-estimate them" (`:932-933`); flag set from the stream event (`codex-rs/core/src/compact.rs:806-808`).
- **Heuristic**: `APPROX_BYTES_PER_TOKEN = 4`, ceiling (`codex-rs/utils/string/src/truncate.rs:4`, `:101-114`), over model-visible bytes only, excluding ids/metadata/JSON escaping (`codex-rs/core/src/context_manager/history.rs:1065-1072`). Encrypted reasoning decoded length `len*3/4 − 650` (`:1053-1059`); encrypted function output `len*9/16` (`:1061-1063`).
- **Images**: flat 7,373 bytes (~1,844 tokens) per image unless `detail: original`, then decode the base64 image and count 32-px patches capped at 10,000, cached in a 32-entry SHA1-keyed LRU (`:1074-1095`, `:1250-1296`); file-referenced originals assume max patches (`:1298-1309`).
- **Full estimate** (diagnostics, token budget) = base instructions (unless already an item) + sum of items (`:649-678`).
- **Thresholds** (`codex-rs/core/src/session/context_window.rs`): `auto_compact_token_limit()` = min(catalog/config, 90 % of window) (`codex-rs/protocol/src/openai_models.rs:527-539`); hard cap `full_context_window_limit = context_window * effective_context_window_percent / 100`, default 95 (`codex-rs/protocol/src/openai_models.rs:391-393`; `context_window.rs:84-86`); scope `BodyAfterPrefix` counts only `active − prefill_input_tokens` and uses the raw config value first (`:67-80`); TokenBudget adds `fallback_buffer_tokens` only when a fallback prompt exists (`:97-103`); post-turn `active*100 >= window_limit*percent` (`:110-116`). Prefill baseline tracked per auto-compact window (`codex-rs/core/src/state/auto_compact_window.rs:34-132`).
- **Config plumbing**: `model_auto_compact_token_limit`, `model_context_window` written into `ModelInfo` (`codex-rs/models-manager/src/model_info.rs:20-31`), window clamped to `max_context_window`; in `Total` scope a user cannot set the threshold above 90 % of the window.
- **Overflow pin**: `set_token_usage_full(context_window)` so the next turn compacts (`codex-rs/core/src/context_manager/history.rs:494-501`).
- **Catalog**: shipped models have `context_window` 272000 (one 372000), `max_context_window` 872000 or equal; unknown-model fallback 272_000 / 95 % (`codex-rs/models-manager/src/model_info.rs:126-133`).

## Constants
| name | value | path:line |
|---|---|---|
| `APPROX_BYTES_PER_TOKEN` | 4 | `codex-rs/utils/string/src/truncate.rs:4` |
| `RESIZED_IMAGE_BYTES_ESTIMATE` | 7373 (~1,844 tokens) | `codex-rs/core/src/context_manager/history.rs:1078` |
| `ORIGINAL_IMAGE_PATCH_SIZE` / `MAX_PATCHES` | 32 px / 10_000 | `codex-rs/core/src/context_manager/history.rs:1082-1086` |
| encrypted reasoning estimate | `len*3/4 − 650` | `codex-rs/core/src/context_manager/history.rs:1053-1059` |
| encrypted function output estimate | `len*9/16` | `codex-rs/core/src/context_manager/history.rs:1061-1063` |
| default auto-compact limit | 90 % of window | `codex-rs/protocol/src/openai_models.rs:530` |
| `effective_context_window_percent` | 95 | `codex-rs/protocol/src/openai_models.rs:392` |
| image patch LRU | 32 entries | `codex-rs/core/src/context_manager/history.rs:1250-1296` |

## Evolution
- 2025-10-20 `049a61bcfc` 90 % auto-compact + `effective_context_window_percent`.
- 2025-10-22 `fd0673e457` "feat: local tokenizer" (tiktoken-rs) → deleted 2025-11-20 `52d0ec4cd8` "Delete tiktoken-rs (#7018)" ([[no-local-tokenizer]]).
- 2025-11-21 `b519267d05` "Account for encrypted reasoning for auto compaction (#7113)".
- 2026-01-15 `1fc72c647f` "Fix token estimate during compaction (#9337)": reasoning length was counted bytes-as-tokens; wrapped in `approx_tokens_from_byte_count` (issue #9287).
- 2026-02-04 `dc7007beaa` (#10692) estimate and request share `base_instructions` → [[summary-estimator-payload-mismatch]].
- 2026-02-11 `40de788c4d` clamp configured limit to 90 % of window → [[auto-compact-threshold-exceeds-window]].

## Versus pi
- [[pi--token-estimation]]: chars/4 (coding-agent) and chars/3.5 (pi-ai) coexist; image 4,800 chars; usage invalidated at compaction boundaries. codex: one bytes/4 estimator, explicit hidden-reasoning add-on, per-image patch math, ratio thresholds with a 5 % hard headroom.
