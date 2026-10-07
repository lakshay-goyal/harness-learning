---
type: failure
concepts: [auto-compaction, thinking-level-abstraction]
harnesses: [pi, opencode]
---
**Symptom** — Compaction requests had a different shape than the session's normal requests and providers rejected or degraded them: non-reasoning models got a `reasoning` parameter; `/compact` forced `reasoning:"high"` (invalid e.g. for Copilot Opus at `medium`); compaction/branch summaries forced `toolChoice:"none"` or exposed tools; Copilot Business summaries used the wrong endpoint.

**Root cause** — The summarizer built its own request options instead of inheriting the session's model capabilities, thinking level and auth/routing.

**Fix · [[pi]]**
- `d35935200` 2026-03-04 (#1793) only pass reasoning when `model.reasoning` (`packages/coding-agent/src/core/compaction/compaction.ts:595-610` at HEAD).
- `da6a81d39` 2026-04-20 (#3438) reuse the session thinking level instead of forcing `high` (originally set in `3c6c9e52c`).
- `6b36eb592` 2026-08-26 (#8649, #8638) no explicit tool choice; 0.84.3 no tools exposed.
- 0.84.0 (#6768, #7579) Copilot Business endpoint/auth for summaries (hash unverified); `35f807cfa` 2026-05-17 route compaction through the session `streamFn`; `_getSummarizationRequestAuth` routes virtual models then resolves auth (`packages/coding-agent/src/core/agent-session.ts:572-609`).

**Fix · [[opencode]]**
- `759635eefa` 2025-11-18 "fix gpt compaction issue": compaction called the model without the session's provider options; GPT/Codex rejected it.
- `4cb29967f6` 2026-03-16 (#17823) plugin `experimental.chat.messages.transform` applied during compaction too, so the summarizer sees what the model normally sees (`packages/opencode/src/session/compaction.ts:378-379`).
- `f9d99f044d` 2026-04-15 (#22371) GitHub Copilot requires a `tools` field when history has tool calls even with no tools enabled → dummy `_noop` tool (`packages/opencode/src/session/llm/request.ts:159-175`).
- `b7f9363393` 2026-08-06 (#40800) replaying the head as provider messages produced invalid shapes on orphaned compaction history → head serialized to flat tagged text.

**Lesson** — Side requests (summaries, classifiers) must inherit the session's model capabilities, auth and transport, and minimize extra knobs.

Related: [[auto-compaction]] · [[thinking-level-abstraction]] · [[summarizer-emits-tool-calls]] · [[summary-output-budget-misfit]]
