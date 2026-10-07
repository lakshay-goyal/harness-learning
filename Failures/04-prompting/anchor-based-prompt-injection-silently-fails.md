---
type: failure
concepts: [dynamic-tool-guidelines, minimal-system-prompt]
harnesses: [codex]
---
**Symptom** — Parallel-tool-call guidance never reached some models: it was spliced into base instructions with `base.replace("## Editing constraints", INSTRUCTIONS)`, and for model prompts lacking that heading the replace was a no-op — the instructions were silently dropped (diff of `4985a7a444` in `dae0608c06:codex-rs/core/src/codex.rs`).

**Root cause** — String-anchor patching of a per-model prompt whose headings differ by model family; no assertion that the anchor exists.

**Fix · [[codex]]** — `4985a7a444` 2025-11-19 "fix: parallel tool call instruction injection (#6893)": append to the model family's instructions instead. Template `80140c6d9d:codex-rs/core/templates/parallel/instructions.md` ("Think first… Batch everything… Use `multi_tool_use.parallel`… Do not try to parallelize using scripting", `4985a7a444:codex-rs/core/templates/parallel/instructions.md`) later deleted by `6382dc2338` 2025-12-09 "enable parallel tc", guidance folded into per-model prompts (`codex-rs/core/gpt_5_2_prompt.md:252`). The same fragility class survives in literal-heading stripping of update_plan guidance (`codex-rs/prompts/src/update_plan_instructions.rs:12-15`).

**Lesson** — Prompt patching by string anchor must assert the anchor exists (or append to a named section); a missing anchor silently ships the old prompt.

Related: [[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[per-model-system-prompt]] · [[codex--dynamic-tool-guidelines|codex]]
