---
type: failure
concepts: [unicode-sanitization, code-mode]
harnesses: [codex]
---
**Symptom** — `await codex.tool(...)` in the `js_repl` kernel hung when a tool response contained a literal U+2028 or U+2029 (`f35d46002a` 2026-03-12).

**Root cause** — The kernel framed messages as JSONL and read them with JavaScript readline. Readline treats U+2028 and U+2029 as line terminators, so a frame was split in the middle of a JSON string. JSON allows these characters unescaped.

**Fix · [[codex]]** — `f35d46002a` 2026-03-12 (#14421), "Fix js_repl hangs on U+2028/U+2029 dynamic tool responses": byte-oriented JSONL framing in `f35d46002a:codex-rs/core/src/tools/js_repl/kernel.js`. `js_repl` itself was later removed (`8a559e7938` 2026-04-24).

**Lesson** — JSON is not newline-safe under JavaScript line-terminator rules. Frame by bytes, or escape U+2028 and U+2029, at every JSONL boundary. This is the same family as [[unpaired-surrogate-breaks-json]]: valid text can still break a framing layer.

Related: [[unicode-sanitization]] · [[code-mode]] · [[unpaired-surrogate-breaks-json]] · [[codex--unicode-sanitization|codex]]
