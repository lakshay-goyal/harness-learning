---
type: failure
concepts: [per-model-system-prompt]
harnesses: [codex]
---
**Symptom** — The model ran slow test/lint suites proactively in interactive sessions, slowing iteration; later wrote tests that mirror the implementation.

**Root cause** — Validation guidance written for autonomous runs applied equally when a human is waiting at each step.

**Fix · [[codex]]** — `3d8bca7814` 2025-08-26 "feat: decrease testing when running interactively (#2707)": "When working in interactive approval modes like **untrusted**, or **on-request**, hold off on running tests or lint commands until the user is ready for you to finalize your output… When working on test-related tasks… you may proactively run tests regardless of approval mode." (`codex-rs/protocol/src/prompts/base_instructions/default.md:155-163`); `ed391d4dd2` 2026-09-03 "Do not write tests for reversible, low-impact changes or that mirror the implementation." (gpt-6-astra catalog text).

**Lesson** — Approval mode is a usable proxy for "is a human waiting"; key validation depth off it.

Related: [[per-model-system-prompt]] · [[approval-policy-modes]] · [[premature-turn-end]] · [[codex--per-model-system-prompt|codex]]
