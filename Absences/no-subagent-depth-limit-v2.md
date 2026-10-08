---
type: absence
harnesses: [codex]
---
# no-subagent-depth-limit-v2

Multi-agent V2 has no nesting depth cap — only concurrency/residency limits.

**What's missing**
- `codex-rs/core/src/tools/spec_plan.rs:726-729`; legacy V1 default max spawn depth 1 (`codex-rs/core/src/config/mod.rs:267`) is ignored by V2.

**Evidence of decision**
- `70ac0f123c` 2026-04-29.

**Implication**
- Recursion is bounded by the model catalog (which models get tools) and the 4-thread cap ([[subagent-concurrency-limits]]); a depth-capped agent once still had tools ([[depth-capped-agent-still-has-tools]]).

Related: [[subagent-concurrency-limits]] · [[in-process-subagent-threads]] · [[depth-capped-agent-still-has-tools]] · [[Absences]]
