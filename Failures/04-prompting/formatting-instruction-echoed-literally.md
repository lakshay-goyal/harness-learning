---
type: failure
concepts: [per-model-system-prompt]
harnesses: [codex]
---
**Symptom** — "the model was often including the literal text "Bold the keyword" in lists" (`4ae45a6c8d` body).

**Root cause** — A terse imperative style rule ("- Bold the keyword, then colon + concise description.") reads like a template to fill.

**Fix · [[codex]]** — `4ae45a6c8d` 2025-09-03 removed the rule; `81b148bda2` (2025-08-07) had already added "Don’t use literal words “bold” or “monospace” in the content." (`codex-rs/protocol/src/prompts/base_instructions/default.md:248`); gpt-5-codex: "avoid naming formatting styles in answers" (`codex-rs/core/gpt_5_codex_prompt.md:57`).

**Lesson** — Terse imperative style rules can be copied as content; phrase them as descriptions of the desired output, not as fill-in templates.

Related: [[per-model-system-prompt]] · [[output-breaks-client-renderer]] · [[codex--per-model-system-prompt|codex]]
