---
type: failure
concepts: [path-normalization, file-read-tool]
harnesses: [pi]
---
**Symptom** — `read` failed with "file not found" on macOS screenshots the user had just dragged in or typed (`Screenshot 2025-… at 10.03.22 AM.png`, French "Capture d'écran …"), and on paths with unicode spaces.

**Root cause** — macOS names contain invisible variants: U+202F narrow NBSP before AM/PM, lowercase am/pm, NFD-decomposed accents, U+2019 curly apostrophe; typed/pasted paths use ASCII space, NFC, straight `'`.

**Fix · [[pi]]**
- `9a7bbb283` 2025-12-13 (#181) — unicode spaces in macOS screenshot names.
- `4edb506df` 2026-01-29 (#1078) — NFD and curly-quote variants, plus combined NFD+curly (`packages/coding-agent/src/core/tools/path-utils.ts:7-20,86-118`).
- `d22c120b8` 2026-04-15 (#3194) — case-insensitive ` am.`/` pm.`.
- Durable copy (`packages/durable/src/tools/path-utils.ts:4-30`) keeps the same variant list.

**Lesson** — User-typed paths and filesystem names differ in invisible characters; on miss, try a small set of normalized variants before failing.

Related: [[path-normalization]] · [[file-read-tool]] · [[pi--path-normalization|pi]]
