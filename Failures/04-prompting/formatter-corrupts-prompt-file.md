---
type: failure
concepts: [per-model-system-prompt]
harnesses: [codex]
---
**Symptom** — The apply_patch grammar shown to the model was wrong: prettier rewrote `***` patch markers in `31d0d7a305:codex-rs/core/prompt.md` into `**_` / `_**` and `\*\*\*` (`31d0d7a305:codex-rs/core/prompt.md`: "**_ Begin Patch"; `e4c275d615` body: "As pointed out in #2030, our prettier formatting started altering prompt.md").

**Root cause** — Prompt files lived in a Markdown-formatted tree; the formatter treated grammar literals as emphasis. The grammar also existed in two copies (prompt + tool definition).

**Fix · [[codex]]** — `a6139aa003` 2025-08-04 restored literal `*** Begin Patch` etc.; `e4c275d615` 2025-08-21 removed the duplicated grammar from `prompt.md` and shipped it only with the tool definition (later Lark grammar; prose instructions deleted `8d637ae398` 2026-08-13).

**Lesson** — Prompt files are code: exclude them from Markdown formatters and keep exactly one copy of any grammar.

Related: [[per-model-system-prompt]] · [[patch-envelope-edit]] · [[tool-description-design]] · [[codex--per-model-system-prompt|codex]]
