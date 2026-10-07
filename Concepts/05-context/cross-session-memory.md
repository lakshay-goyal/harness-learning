---
type: concept
stage: context
tier: variant
aliases: [memories, memory_summary.md, MEMORY.md, raw_memories.md, rollout_summaries, stage_one, stage1_outputs, phase2 consolidation, read_path.md, oai-mem-citation, memory_mode, MemoryVersion, ad_hoc notes]
harnesses: [codex]
---
A background LLM pipeline distils finished sessions into a persistent memory folder; a bounded summary is injected into every new session's developer instructions and the model reads details on demand, with citations feeding usage-ranked retention.

## Why
- Each new session otherwise re-learns the user's repos, preferences and past fixes from scratch.
- An always-injected memory amplifies every mistake: overgeneralized preferences ([[memory-overgeneralizes-preferences]]), stale facts presented as current ([[stale-memory-presented-as-current]]), and noise from trivial sessions ([[memory-noise-without-noop]]).
- Transcripts contain third-party content and secrets → extraction must treat them as data and redact.
- Cost must be bounded: background, few rollouts per startup, rate-limit floor, cheap reasoning.

## Design space
- No cross-session memory; only instruction files written by the user ✔ pi ([[context-file-hierarchy]]).
- **Two-phase LLM pipeline**: per-rollout extraction (structured JSON) → global consolidation sub-agent editing a git-baselined memory folder ✔ codex.
- What to record: "task history, not a user profile" (codex v2) vs preference-centric profile (codex v1 2026-03 → 2026-09).
- Injection: bounded summary (2,500 tokens) in developer instructions + on-demand reads via shell or dedicated tools ✔ codex.
- Retention: usage-ranked (citations + shell-path parsing), unused-days expiry ✔ codex.
- Writes by the model: only on explicit user request, as ad-hoc note files ✔ codex.
- Pollution guard: threads using external context marked `polluted` and excluded ✔ codex.
- Prior art inside codex: live memory editing tried and disabled (`58ac2a8773`); memories MCP added and dropped (`d579dafb70`).

## Implementations
- [[codex--cross-session-memory|codex]] — `codex-rs/memories/{read,write}` + `codex-rs/ext/memories`; phase 1 Low effort, 8-way concurrency; phase 2 consolidation agent with no approvals/network; `memory_summary.md` ≤2,500 tokens injected; `<oai-mem-citation>` usage tracking.

## Failures
- [[memory-overgeneralizes-preferences]]
- [[stale-memory-presented-as-current]]
- [[memory-noise-without-noop]]

## Related
[[context-file-hierarchy]] · [[model-requested-context-reset]] · [[auxiliary-model-calls]] · [[sqlite-session-index]] · [[delegate-session-runner]] · [[secret-handling]] · [[no-prompt-injection-defense]]
