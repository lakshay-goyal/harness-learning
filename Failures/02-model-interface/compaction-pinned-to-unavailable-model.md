---
type: failure
concepts: [model-catalog, overflow-recovery, auto-compaction, model-resolution]
harnesses: [codex]
---
**Symptom** — Pre-sampling compaction deliberately uses the *previous* turn's model in two cases: the compaction-compatibility hash (`comp_hash`) changed, or the user downshifted to a smaller-window model. On resumed ChatGPT threads, the old model slug was sometimes no longer served. The backend rejected the compaction request, and the next turn was blocked (`172ab264bd` body).

**Root cause** — The history pins a model identity, and the vendor's catalog moves on. "Summarize with the model that produced the history" had no fallback for a retired model.

**Fix · [[codex]]**
- `172ab264bd` 2026-07-07 (#30319), "retry rejected previous-model compaction with selected model": retry once with the currently selected model (`codex-rs/core/src/compact_model_fallback.rs:10-32`). Conditions:
  - the provider is OpenAI with Codex-backend or API-key auth;
  - the models or access programs differ;
  - the error is not an abort, interrupt or budget error.
- Counted by the telemetry counter `codex.compaction.model_fallback` (`codex-rs/core/src/compact_model_fallback.rs:57-65`).
- The remote v2 path calls the same fallback (`codex-rs/core/src/compact_remote_v2.rs:244-283`).
- Related shape fixes:
  - `5f3180c793` 2026-09-25: restore the previous turn's `cyber_access_program`, so the model/program pair is accepted (`codex-rs/core/src/session/turn.rs:1365-1371`).
  - `35d9e4bc4d` 2026-09-08: reuse the pinned effort.

  See [[compaction-request-shape-mismatch]].

**Lesson** — Anything that replays the model recorded in history must tolerate that model having been retired, and needs a defined fallback to the current model.

Related: [[model-catalog]] · [[model-resolution]] · [[auto-compaction]] · [[overflow-recovery]] · [[cross-provider-handoff]] · [[codex--model-resolution|codex]] · [[codex--model-catalog|codex catalog]] · [[compaction-request-shape-mismatch]]
