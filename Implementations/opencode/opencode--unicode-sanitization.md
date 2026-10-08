---
type: implementation
harness: opencode
concept: unicode-sanitization
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:25-27, packages/opencode/src/provider/transform.ts:101-166]
---
[[unicode-sanitization]] in [[opencode]].

## Mechanism
- Legacy: `sanitizeSurrogates(content)` replaces unpaired UTF-16 surrogates with `U+FFFD` (`packages/opencode/src/provider/transform.ts:25-27`).
- Applied in `normalizeMessages` to every string message content, text part and text/error-text tool-result output, as AI SDK middleware on the final prompt (`transform.ts:101-166`).
- v2 (`packages/llm`, `packages/core`): no request-content surrogate sanitization: grep for surrogate handling in `packages/llm/src` and `packages/core/src` finds only the ripgrep line-cap trim of a dangling high surrogate (`packages/core/src/ripgrep.ts:269`).

## Constants
none.

## Evolution
- 2026-05-05 `6409aceb1a` "fix: sanitize surrogates" (#25934).

Failures: [[unpaired-surrogate-breaks-json]].

Contrast: [[pi--unicode-sanitization|pi]] uses the same `sanitizeSurrogates` name and placement (all text before serialization); opencode added it ~a year after launch.
