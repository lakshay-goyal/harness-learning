---
type: failure
concepts: [search-tools, tool-call-gate]
harnesses: [pi]
---
**Symptom** — A model-supplied `grep`/`find` pattern beginning with `-` (e.g. `--pre=cmd`) was parsed by ripgrep/fd as a **flag** — `--pre` makes rg execute a command per file (argument injection).

**Root cause** — The pattern was passed positionally without an end-of-options marker.

**Fix · [[pi]]** — `3d43d2e17` 2026-04-30 (#4018): insert `--` before pattern/path (`packages/coding-agent/src/core/tools/grep.ts:162-166`, `find.ts:182-214`).

**Lesson** — Every argv built from model input needs `--` before untrusted positionals; spawning without a shell is not enough.

Related: [[search-tools]] · [[tool-call-gate]] · [[no-sandbox]] · [[pi--search-tools|pi]]
