---
type: failure
concepts: [per-model-system-prompt]
harnesses: [opencode, codex]
---
**Symptom** — gpt.txt (taken from the Codex desktop app) required "markdown links (not inline code) for clickable file paths" with absolute filesystem targets like `[app.ts](/abs/path/app.ts)`; the opencode TUI rendered these poorly ("file reference annoyances").

**Root cause** — Output-format rules written for another client's renderer.

**Fix · [[opencode]]** — `72cb9dfa31` 2026-03-28 "adjust gpt prompt to be more minimal, fix file reference annoyances": link rule removed; HEAD "Use inline code blocks for commands, paths…" (`packages/opencode/src/session/prompt/gpt.txt:75`). codex.txt keeps "Use inline code to make file paths clickable … Do not use URIs like file://" (`codex.txt:73-77`).

**Lesson** — Output-format rules must match the client renderer they ship with.

Related: [[per-model-system-prompt]] · [[borrowed-prompt-foreign-references]] · [[opencode--per-model-system-prompt|opencode]]

## Also: [[codex]] (folded from `output-breaks-client-renderer`)
**Symptom** — Fenced code blocks without an info string "causes auto-language detection in the extension to incorrectly highlight the code" (`35c76ad47d` body); file references not clickable (line ranges, `file://` URIs); raw ANSI codes in answers.

**Root cause** — The final answer is parsed by several clients (TUI, IDE extension); model defaults don't match their parsers.

**Fix · [[codex]]** — `35c76ad47d` 2025-10-01 "include an info string as often as possible"; `b1c291e2bb` / `d60cbed691` 2025-09-15 **File References** block ("Do not use URIs like file://, vscode://, or https://.", "Do not provide range of lines", `codex-rs/protocol/src/prompts/base_instructions/default.md:219-227`); "Don’t output ANSI escape codes directly — the CLI renderer applies them." (`default.md:250`).

**Lesson** — The final-answer spec in the prompt is a renderer contract; every client parsing quirk shows up as a prompt line.

Related: [[per-model-system-prompt]] · [[formatting-instruction-echoed-literally]] · [[foreign-harness-tool-hallucination]] · [[codex--per-model-system-prompt|codex]]
