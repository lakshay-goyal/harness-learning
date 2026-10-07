---
type: failure
concepts: [cross-provider-handoff, image-normalization]
harnesses: [codex]
---
**Symptom** — Mid-session model switches replayed history the new model could not accept:
- Images with `detail: original` were sent to a model that does not support it (`6eecd04fc1` 2026-09-09).
- Model-switch compaction kept stale `<model_switch>` base instructions (`67e577da53` 2026-02-13).
- Spawned agents could request models from a different multi-agent backend (`92938d880e` 2026-07-13).

**Root cause** — History is written for the model that produced it. Replay copied it verbatim instead of projecting it onto the receiving model's capabilities.

**Fix · [[codex]]**
- `6eecd04fc1` 2026-09-09, "Normalize image detail for the receiving model": request copies downgrade `original` → `high`.
- `5e01450963` 2026-02-10 (#11349): strip unsupported images and audio, using the placeholder "image content omitted because you do not support image input" (`codex-rs/core/src/context_manager/normalize.rs:330-420`; `codex-rs/core/src/context/unsupported_media.rs:11-19`). See [[image-content-poisoning]].
- `67e577da53` 2026-02-13: drop stale model-switch instructions on compaction.
- `92938d880e` 2026-07-13: keep spawned agents on the parent's backend.

**Lesson** — History is written for one model. Every replay must be re-projected onto the receiving model's capabilities (modalities, detail levels, instructions), on the request copy, leaving raw history untouched.

Related: [[cross-provider-handoff]] · [[image-normalization]] · [[transcript-replay-repair]] · [[codex--cross-provider-handoff|codex]] · [[truncation-budget-drift-on-replay]] · [[image-content-poisoning]]
