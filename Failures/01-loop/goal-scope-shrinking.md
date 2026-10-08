---
type: failure
concepts: [persistent-goal-continuation]
harnesses: [codex]
---
**Symptom** — Under a persistent objective the model declared success on a smaller or merely passing subset, substituting easier-to-test solutions (`96836e15ed` body).

**Fix · [[codex]]**
- `96836e15ed` 2026-05-11 — Fidelity section ("Do not substitute a narrower, safer, smaller, merely compatible, or easier-to-test solution because it is more likely to pass current tests.") and Completion audit ("The audit must prove completion, not merely fail to find obvious remaining work.") (`codex-rs/ext/goal/templates/goals/continuation.md:30-46`); "do not redefine success around a smaller or easier task".

**Lesson** — Long-horizon loops need an adversarial completion audit mapping every requirement to authoritative evidence.

Related: [[persistent-goal-continuation]] · [[goal-loop-premature-stop]] · [[codex--persistent-goal-continuation|codex]]
