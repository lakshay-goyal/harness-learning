---
type: implementation
harness: codex
concept: cross-provider-handoff
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:942-953, codex-rs/core/src/context_manager/normalize.rs:330-420, codex-rs/core/src/context/unsupported_media.rs:11-19, codex-rs/core/src/session/mod.rs:4370-4375, codex-rs/core/src/session/turn.rs:1337-1431, codex-rs/core/src/session/turn.rs:1365-1371, codex-rs/core/src/compact_model_fallback.rs:10-65]
---
[[cross-provider-handoff]] in [[codex]]. The scope is narrow: within one wire protocol (Responses), handoff is between *models* and between OpenAI and Responses-compatible third parties. It is not between vendor formats.

## Mechanism
- **Project history per receiving model at request time.** Raw history is untouched; `for_prompt` / `normalize_history` run on a clone ([[codex--transcript-replay-repair|codex replay repair]]).
  - Images and audio unsupported by the current model's `input_modalities` are replaced with the text "image content omitted because you do not support image input" (`codex-rs/core/src/context/unsupported_media.rs:11-19`; `codex-rs/core/src/context_manager/normalize.rs:330-420`).
  - Image `detail: original` is downgraded to `high` on request copies for models that don't support original detail (`6eecd04fc1` 2026-09-09, "Normalize image detail for the receiving model").
  - Image detail is stripped for Responses Lite models (`4435ff2810` 2026-06-10).
- **Third-party Responses providers.** Internal chat-message metadata passthrough is stripped and `encrypted_function_args` is cleared on `FunctionCall` items (`codex-rs/core/src/client.rs:942-953`). Encrypted reasoning is sent unchanged to everyone, with no same-model check.
- **Model-switch notice.** A `<model_switch>` fragment is inserted first in the developer bundle when the model changes (`codex-rs/core/src/session/mod.rs:4370-4375`).
- **Summarize with the model that wrote the history.** `maybe_run_previous_model_inline_compact` compacts with the *previous* model in two cases (`codex-rs/core/src/session/turn.rs:1337-1431`):
  - the compaction-compatibility hash `comp_hash` changed;
  - the switch is to a smaller window and current usage exceeds the new limit.

  The previous turn's `cyber_access_program` is restored, so the server does not reject the model/program pair (`codex-rs/core/src/session/turn.rs:1365-1371`). If the old model is rejected (e.g. retired), compaction retries with the current model (`codex-rs/core/src/compact_model_fallback.rs:10-65`).
- **Truncation budget pinned to the originating model.** Each tool output persists `history_truncation_token_limit`, so replay under another model neither expands nor shrinks history (`aa88a0333c` 2026-09-09; [[truncation-budget-drift-on-replay]]).

## Evolution
- `5e01450963` 2026-02-10 (#11349): strip unsupported images from prompt history to guard against a model switch.
- `67e577da53` 2026-02-13: model-switch compaction kept stale `<model_switch>` base instructions; fixed.
- `4435ff2810` 2026-06-10: strip image detail for Lite.
- `172ab264bd` 2026-07-07 (#30319): retry a rejected previous-model compaction with the selected model.
- `92938d880e` 2026-07-13: spawned agents could request models from a different multi-agent backend; fixed.
- `aa88a0333c` / `6eecd04fc1` 2026-09-09: originating truncation budget persisted; image detail normalized for the receiving model.
- `5f3180c793` 2026-09-25 (#48224): the previous turn's access program restored for previous-model compaction.

## Versus pi
- [[pi--cross-provider-handoff|pi]] `transformMessages` rewrites across vendors: it demotes foreign reasoning to text, reshapes ids and handles signature requirements.
- Codex never crosses wire formats. Its handoff problems are *capability* differences between models of one family: modalities, image detail, window size, compaction compatibility.

## Failures
- [[model-switch-replays-unsupported-content]]
- [[compaction-pinned-to-unavailable-model]]
- [[truncation-budget-drift-on-replay]]
- [[image-content-poisoning]]
