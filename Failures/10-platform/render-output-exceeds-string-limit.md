---
type: failure
concepts: [differential-tui-rendering]
harnesses: [pi]
---
**Symptom** — Image-heavy or very large frames crashed the TUI with V8 "Invalid string length" (#8028).

**Root cause** — The renderer concatenated the whole frame (incl. inline image escape payloads) into one string before writing.

**Fix · [[pi]]** — `6c4f36026` 2026-08-26: `BoundedTerminalWriter` streams output in `MAX_RENDER_WRITE_CHARS` = 1 MiB chunks (`packages/tui/src/tui-main-screen.ts:9-18`).

**Lesson** — Never build unbounded strings for terminal output; stream bounded chunks.

Related: [[differential-tui-rendering]] · [[image-normalization]] · [[pi--differential-tui-rendering|pi]]
