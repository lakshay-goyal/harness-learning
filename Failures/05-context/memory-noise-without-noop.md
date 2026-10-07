---
type: failure
concepts: [cross-session-memory]
harnesses: [codex]
---
**Symptom** — Early memory extraction wrote memories for every rollout (generic status, one-off queries); PR `ac66252f50` lists missing guidance on "when to no-op".

**Root cause** — The extraction prompt had no explicit, preferred empty output, so the model always produced something.

**Fix · [[codex]]** — `ac66252f50` 2026-02-12 "fix: update memory writing prompt (#11546)" ("The previous prompts were less explicit about: when to no-op, schema of the output, how to triage task outcomes, how to distinguish durable signal from noise, and how to consolidate incrementally without churn."): "**No-op is allowed and preferred**" with exact empty JSON `{"rollout_summary":"","rollout_slug":"","raw_memory":""}` and gate "Will a future agent plausibly act better because of what I write here?" (`codex-rs/memories/write/templates/memories/stage_one_system.md:26-47`, `:33-34`).

**Lesson** — Give extractors an explicit, preferred no-op output and a minimum-signal gate.

Related: [[cross-session-memory]] · [[codex--cross-session-memory|codex]]
