---
type: failure
concepts: [subagent-concurrency-limits]
harnesses: [codex]
---
**Symptom** — An agent at max depth still received the multi-agent spawn tools; every spawn then failed deterministically.

**Fix · [[codex]]**
- `352f37db03` 2026-03-26 "fix: max depth agent still has v2 tools"; at HEAD V1 collaboration tools are not registered past max depth (`codex-rs/core/src/tools/spec_plan.rs:719-731`); call-time error remains "Agent depth limit reached. Solve the task yourself." (`codex-rs/core/src/tools/handlers/multi_agents/spawn.rs:73-79`). V2 dropped the depth cap entirely and gates tools by model catalog capability (`codex-rs/core/src/tools/spec_plan.rs:726-729`; `70ac0f123c`).

**Lesson** — Hide tools that will deterministically fail instead of rejecting at call time.

Related: [[subagent-concurrency-limits]] · [[prompt-names-unavailable-tools]] · [[no-subagent-depth-limit-v2]] · [[codex--subagent-concurrency-limits|codex]]
