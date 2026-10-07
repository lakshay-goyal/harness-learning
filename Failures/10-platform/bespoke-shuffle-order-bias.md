---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — A custom seeded shuffle (`seed: 42`) for comparative evals added machinery that could interact with comparison direction (order effects between baseline/candidate arms).

**Root cause** — Bespoke randomization instead of a design that cancels order bias by construction.

**Fix · [[pi]]** — `743e1595c` 2026-07-30 removed custom shuffling (Vitest built-in shuffle); HEAD `createTaskPlan` alternates arm order per repetition — odd runs without→with, even runs with→without "to reduce order bias" (`packages/evals/src/plan.ts:51-56`; `packages/evals/README.md:80`).

**Lesson** — Prefer simple counterbalanced ordering over bespoke RNG for paired A/B evals.

Related: [[harness-evals]] · [[pi--harness-evals|pi]]
