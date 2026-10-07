---
type: failure
concepts: [auto-compaction, thinking-level-abstraction]
harnesses: [pi, codex]
---
**Symptom** — Compaction requests had a different shape than the session's normal requests and providers rejected or degraded them: non-reasoning models got a `reasoning` parameter; `/compact` forced `reasoning:"high"` (invalid e.g. for Copilot Opus at `medium`); compaction/branch summaries forced `toolChoice:"none"` or exposed tools; Copilot Business summaries used the wrong endpoint.

**Root cause** — The summarizer built its own request options instead of inheriting the session's model capabilities, thinking level and auth/routing.

**Fix · [[pi]]**
- `d35935200` 2026-03-04 (#1793) only pass reasoning when `model.reasoning` (`packages/coding-agent/src/core/compaction/compaction.ts:595-610` at HEAD).
- `da6a81d39` 2026-04-20 (#3438) reuse the session thinking level instead of forcing `high` (originally set in `3c6c9e52c`).
- `6b36eb592` 2026-08-26 (#8649, #8638) no explicit tool choice; 0.84.3 no tools exposed.
- 0.84.0 (#6768, #7579) Copilot Business endpoint/auth for summaries (hash unverified); `35f807cfa` 2026-05-17 route compaction through the session `streamFn`; `_getSummarizationRequestAuth` routes virtual models then resolves auth (`packages/coding-agent/src/core/agent-session.ts:572-609`).

**Fix · [[codex]]**
- Symptom variant: compaction used the newly selected reasoning effort while sampling still used an earlier pinned effort, and the old pin survived into the new window; model-switch compaction paired the *previous* model with the *current* access program, which the server rejects; previous-model compaction rejected outright when that model was retired.
- `35d9e4bc4d` 2026-09-08 (#43796) reasoning-effort pin vs compaction.
- `5f3180c793` 2026-09-25 (#48224) restore the previous turn's `cyber_access_program` for previous-model compaction (`codex-rs/core/src/session/turn.rs:1365-1371`).
- `172ab264bd` 2026-07-07 (#30319) retry a rejected previous-model compaction once with the current model (`codex-rs/core/src/compact_model_fallback.rs:13-36`) → [[compaction-pinned-to-unavailable-model]].
- Compaction reuses one `ModelClientSession` so sticky routing / websocket state match the turn (`codex-rs/core/src/compact.rs:282-284`); deferred tools filtered for compaction requests too (`85c1500569`, [[deferred-tools-lost-on-resume]]).

**Lesson** — Side requests (summaries, classifiers) must replay the exact request shape of the turn they serve — model capabilities, effort pin, auth/program pairing, transport, tool visibility — with a fallback when that shape is rejected.

Related: [[auto-compaction]] · [[thinking-level-abstraction]] · [[summarizer-emits-tool-calls]] · [[summary-output-budget-misfit]] · [[codex--auto-compaction|codex]]
