---
type: failure
concepts: [shell-execution, tool-output-spill]
harnesses: [opencode]
---
**Symptom** — The model piped commands through `head`/`tail` to limit output, losing the lines it later needed, even though the harness already kept the full output in a file.

**Root cause** — The model did not know the harness truncates and spills output itself.

**Fix · [[opencode]]**
- `1b82511fbd` 2026-01-07 truncated output written to a file.
- `e79d41c70e` 2026-03-03 "clarify output capture guidance": "If the output exceeds ${maxLines} lines or ${maxBytes} bytes, it will be truncated and the full output will be written to a file … Do NOT use `head`, `tail`, or other truncation commands to limit output" (`packages/opencode/src/tool/shell/prompt.ts:98`).

**Lesson** — Tell the model the harness already captures everything and where it goes.

Related: [[shell-execution]] · [[tool-output-spill]] · [[tool-output-truncation]] · [[tool-description-design]] · [[opencode--shell-execution|opencode]]
