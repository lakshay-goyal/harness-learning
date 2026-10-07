---
type: failure
concepts: [per-model-system-prompt]
harnesses: [codex]
---
**Symptom** — Fenced code blocks without an info string "causes auto-language detection in the extension to incorrectly highlight the code" (`35c76ad47d` body); file references not clickable (line ranges, `file://` URIs); raw ANSI codes in answers.

**Root cause** — The final answer is parsed by several clients (TUI, IDE extension); model defaults don't match their parsers.

**Fix · [[codex]]** — `35c76ad47d` 2025-10-01 "include an info string as often as possible"; `b1c291e2bb` / `d60cbed691` 2025-09-15 **File References** block ("Do not use URIs like file://, vscode://, or https://.", "Do not provide range of lines", `codex-rs/protocol/src/prompts/base_instructions/default.md:219-227`); "Don’t output ANSI escape codes directly — the CLI renderer applies them." (`default.md:250`).

**Lesson** — The final-answer spec in the prompt is a renderer contract; every client parsing quirk shows up as a prompt line.

Related: [[per-model-system-prompt]] · [[formatting-instruction-echoed-literally]] · [[foreign-harness-tool-hallucination]] · [[codex--per-model-system-prompt|codex]]
