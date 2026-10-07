---
type: failure
concepts: [per-model-system-prompt]
harnesses: [opencode]
---
**Symptom** — gpt.txt (taken from the Codex desktop app) required "markdown links (not inline code) for clickable file paths" with absolute filesystem targets like `[app.ts](/abs/path/app.ts)`; the opencode TUI rendered these poorly ("file reference annoyances").

**Root cause** — Output-format rules written for another client's renderer.

**Fix · [[opencode]]** — `72cb9dfa31` 2026-03-28 "adjust gpt prompt to be more minimal, fix file reference annoyances": link rule removed; HEAD "Use inline code blocks for commands, paths…" (`packages/opencode/src/session/prompt/gpt.txt:75`). codex.txt keeps "Use inline code to make file paths clickable … Do not use URIs like file://" (`codex.txt:73-77`).

**Lesson** — Output-format rules must match the client renderer they ship with.

Related: [[per-model-system-prompt]] · [[borrowed-prompt-foreign-references]] · [[opencode--per-model-system-prompt|opencode]]
