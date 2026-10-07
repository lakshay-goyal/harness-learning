---
type: failure
concepts: [constrained-tool-sampling, search-replace-edit, patch-envelope-edit]
harnesses: [codex]
---
**Symptom**
- The freeform `apply_patch` tool (Lark grammar, Responses "custom" tool) was introduced in `236c4f76a6` 2025-08-22. Two days later it showed "issues… let's disable by default until it stabilizes" (`4157788310` 2025-08-24).
- A grammar bug needed `6f75114695` 2025-09-02 "[apply-patch] Fix lark grammar".
- Later, the `js_repl` freeform tool needed grammar blocks for wrapped payload prefixes (`73fd939296` 2026-02-20).

**Root cause** — With grammar-constrained decoding, the grammar *is* the tool's input contract. A grammar bug blocks every invocation, or skews the model's output for every edit. Model-side training on the format also has to catch up.

**Fix · [[codex]]**
- Staged rollout: disabled by default (`4157788310`), the grammar fixed (`6f75114695`), and re-enabled as the constrained default in `4764fc1ee7` 2025-10-04.
- Today the grammar is a 19-line asset, `codex-rs/core/assets/tools/apply_patch.lark`, compiled in (`codex-rs/core/src/tools/handlers/apply_patch_spec.rs:5-27`).
- The JSON variant was deleted in `e783341b70` 2026-05-08.
- `js_repl` was removed altogether (`8a559e7938` 2026-04-24).

**Lesson** — Grammar-constrained tool formats need a staged rollout behind a flag, with grammar tests, because a grammar bug is a total outage of the tool, not a degraded case.

Related: [[constrained-tool-sampling]] · [[patch-envelope-edit]] · [[codex--constrained-tool-sampling|codex]] · [[malformed-tool-json-crashes]] · [[edit-format]]
