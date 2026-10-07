---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — A vitest `expect` inside the harness made "the eval setup is broken" indistinguishable from "the model did badly".

**Root cause** — Infrastructure invariants and graded outcomes shared one failure channel.

**Fix · [[pi]]** — `a0ac81c0a` 2026-07-26: invariants via node `assert`, scored expectations via `expect`. Later doctrine: `judgeThreshold: null` — a low score is data, vitest assertions only for broken suite invariants (`packages/evals/README.md:129`); vitest `failed` → `errored`, never 0 (`src/report.ts:101-106`); reported provider/model ≠ planned → `errored` (`report.ts:161-162`).

**Lesson** — Separate infrastructure invariants from graded outcomes; errors are not zero scores.

Related: [[harness-evals]] · [[pi--harness-evals|pi]]
