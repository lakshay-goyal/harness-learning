---
type: failure
concepts: [auto-compaction, turn-lifecycle-hooks]
harnesses: [pi]
---
**Symptom** — A large tool result pushed context over the threshold, but the follow-up provider request was sent before compaction ran → overflow; long tool loops grew past the window between user prompts.

**Root cause** — The threshold was checked only at prompt time and after the whole run (`agent_end`). History: `5a9d844f9` (2025-12-09) had removed "proactive compaction (aborting mid-turn when threshold approached)", leaving no mid-run check.

**Fix · [[pi]]** — `56700d42e` 2026-08-28 (#8782, closes #6879) "compact before post-tool model requests": `prepareNextTurn` now runs only when another turn actually happens (after `shouldStopAfterTurn`), and AgentSession installs `_compactBeforeNextAssistantResponse` there (`packages/coding-agent/src/core/agent-session.ts:776-785`, `898-906`; loop `packages/agent/src/agent-loop.ts:184-189`). Virtual models re-check against the routed model in `prepareRequest` (`agent-session.ts:836-841`). Earlier 0.23.3 added the pre-prompt check.

**Lesson** — The compaction trigger belongs at "before every provider request", not "after each user turn" — non-destructively at turn boundaries rather than by aborting mid-turn.

Related: [[auto-compaction]] · [[turn-lifecycle-hooks]] · [[overflow-recovery]] · [[oversized-trailing-tool-results-uncompactable]]
