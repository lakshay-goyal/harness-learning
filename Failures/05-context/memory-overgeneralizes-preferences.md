---
type: failure
concepts: [cross-session-memory]
harnesses: [codex]
---
**Symptom** — After `11812383c5` (2026-03-12) told Phase 1 that preference evidence is valuable "even when Phase 1 cannot yet tell whether the preference is globally stable", memories turned single-task instructions into durable user-profile rules that were injected into every new session.

**Root cause** — Always-injected memory + an extractor rewarded for recording preferences ("Optimize for future user time saved", "read much more into user messages than assistant messages").

**Fix · [[codex]]** — `74d3a5bf10` 2026-09-08 "Add summary-only extraction for memory v2 (#43800)" + `553df1c691` (#43813): "Write task history, not a user profile." with the counter-example — user said "show me the plan before editing this" → record "the user asked to show a plan before editing", NOT "the user prefers the agent to show plans before editing" (`codex-rs/memories/write/templates/memories/stage_one_system_v2.md:22-31`, `:45`); consolidation_v2 "Keep single-task requests, choices, decisions, and corrections with their task… Ordinary behavior is not a personal preference." and "Overly broad or rigid rules inferred from past tasks can therefore mislead future agents".

**Lesson** — An always-injected memory amplifies every overgeneralization; the extractor must record scope and provenance, not traits.

Related: [[cross-session-memory]] · [[stale-memory-presented-as-current]] · [[codex--cross-session-memory|codex]]
