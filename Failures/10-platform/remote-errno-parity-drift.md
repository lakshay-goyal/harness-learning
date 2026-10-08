---
type: failure
concepts: [remote-execution-env, harness-evals]
harnesses: [pi]
---
**Symptom** — The remote env returned different error semantics than the local Node env on Windows: listing a regular file gave `ENOENT` instead of `ENOTDIR`; Durable env tests failed on Windows (`flushFile` on a directory, Git Bash `$PWD`, taskkill timing).

**Root cause** — Native OS errors weren't translated the way libuv/Node translate them, while the contract demands Node-identical results.

**Fix · [[pi]]** — `956e81504` 2026-10-05: map Windows `ERROR_DIRECTORY` (267) → `ENOTDIR` like libuv; `1543dd8f6` 2026-10-04: Windows env test fixes. Guarded by Durable's env conformance suite + random-sequence differential test Remote vs Node (`packages/env/README.md:40-43`; `packages/durable/src/testing/env-conformance.ts`).

**Lesson** — When an alternative backend promises reference semantics, mirror the reference's error translation and enforce it with shared conformance + differential tests.

Related: [[remote-execution-env]] · [[harness-evals]] · [[pluggable-tool-backends]] · [[pi--remote-execution-env|pi]]
