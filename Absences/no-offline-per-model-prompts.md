---
type: absence
harnesses: [codex]
---
# no-offline-per-model-prompts

No hard-coded per-family prompt selection in the client; the prompt for a model is whatever the model catalog (bundled `models.json` or remote) says.

**What's missing**
- Five `codex-rs/core/gpt*_prompt.md` files remain but are orphaned (no references).

**Evidence of decision**
- Before 2026-02-09 selection was a `starts_with` chain in `find_model_info_for_slug` (`a1abd53b6a^:codex-rs/core/src/models_manager/model_info.rs:115-310`); 2026-02-09 `a1abd53b6a` "Remove offline fallback for models (#11238)" removed all `include_str!` per-model prompts.

**Implication**
- Prompt ownership moved to the catalog ([[model-catalog]], [[per-model-system-prompt]], [[single-vs-per-model-system-prompt]]); unknown models fall back to generic defaults (272_000 window, 10_000-byte tool output, `codex-rs/models-manager/src/model_info.rs:126-132`).

Related: [[per-model-system-prompt]] · [[model-catalog]] · [[single-vs-per-model-system-prompt]] · [[Absences]]
