---
type: implementation
harness: codex
concept: unicode-sanitization
commit: 622e9e3696
files: [codex-rs/core/src/exec.rs:809-811, codex-rs/core/src/unified_exec/async_watcher.rs:468]
---
[[unicode-sanitization]] in [[codex]]. There is no sanitizer at the provider boundary; it is not needed, because the type system enforces valid text.

## Mechanism
- **Producer-side lossy decoding.** Process output is decoded with `from_utf8_lossy`, which replaces invalid bytes with U+FFFD:
  - one-shot exec stdout, stderr and aggregated output (`codex-rs/core/src/exec.rs:809-811`);
  - the unified-exec transcript (`codex-rs/core/src/unified_exec/async_watcher.rs:468`).
- **Language invariant.** Rust `String` is always valid UTF-8, so the request serializer can never meet an unpaired UTF-16 surrogate. The JavaScript-side failure mode ([[unpaired-surrogate-breaks-json]]) cannot occur in the Rust core.
- **Remaining hazard: framing, not encoding.** Text that is valid Unicode can still break line-delimited protocols. U+2028 and U+2029 inside JSONL frames hung the JavaScript `js_repl` kernel, whose readline treated them as line breaks ([[line-separator-breaks-jsonl-framing]]).

## Evolution
- `f35d46002a` 2026-03-12 (#14421): "Fix js_repl hangs on U+2028/U+2029 dynamic tool responses". Framing became byte-oriented in `f35d46002a:codex-rs/core/src/tools/js_repl/kernel.js`.
- `8a559e7938` 2026-04-24 (#19410): `js_repl` removed entirely; superseded by V8 code mode.

## Versus pi
- [[pi--unicode-sanitization|pi]] (TypeScript) must strip lone surrogates in every message converter.
- Codex sanitizes once, when bytes become `String`. The language rules out the invalid state, so only framing bugs at JavaScript boundaries remain.

## Failures
- [[line-separator-breaks-jsonl-framing]]
