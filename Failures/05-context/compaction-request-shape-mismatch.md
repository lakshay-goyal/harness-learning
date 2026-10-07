---
type: failure
concepts: [auto-compaction, thinking-level-abstraction]
harnesses: [pi]
---
**Symptom** — Compaction requests had a different shape than the session's normal requests and providers rejected or degraded them: non-reasoning models got a `reasoning` parameter; `/compact` forced `reasoning:"high"` (invalid e.g. for Copilot Opus at `medium`); compaction/branch summaries forced `toolChoice:"none"` or exposed tools; Copilot Business summaries used the wrong endpoint.

**Root cause** — The summarizer built its own request options instead of inheriting the session's model capabilities, thinking level and auth/routing.

**Fix · [[pi]]**
- `d35935200` 2026-03-04 (#1793) only pass reasoning when `model.reasoning` (`packages/coding-agent/src/core/compaction/compaction.ts:595-610` at HEAD).
- `da6a81d39` 2026-04-20 (#3438) reuse the session thinking level instead of forcing `high` (originally set in `3c6c9e52c`).
- `6b36eb592` 2026-08-26 (#8649, #8638) no explicit tool choice; 0.84.3 no tools exposed.
- 0.84.0 (#6768, #7579) Copilot Business endpoint/auth for summaries (hash unverified); `35f807cfa` 2026-05-17 route compaction through the session `streamFn`; `_getSummarizationRequestAuth` routes virtual models then resolves auth (`packages/coding-agent/src/core/agent-session.ts:572-609`).

**Lesson** — Side requests (summaries, classifiers) must inherit the session's model capabilities, auth and transport, and minimize extra knobs.

Related: [[auto-compaction]] · [[thinking-level-abstraction]] · [[summarizer-emits-tool-calls]] · [[summary-output-budget-misfit]]
