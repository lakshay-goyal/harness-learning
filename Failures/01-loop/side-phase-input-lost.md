---
type: failure
concepts: [steering-queue, follow-up-queue, auto-compaction]
harnesses: [pi]
---
**Symptom** — Messages typed during compaction, branch summarization or tree navigation were dropped, stuck in the queue, wiped the editor, or failed; queued steering turned into follow-ups (or vice versa) across compaction.

**Root cause** — Non-LLM long phases had no input queue or no drain step afterwards; each phase re-implemented input handling.

**Fix · [[pi]]** (≥9 fixes, recurring class)
- 0.17.0 blocked input during compaction (`packages/coding-agent/CHANGELOG.md:5791`); 0.37.0 queue during compaction (#476).
- 0.52.7 resume loop when Agent queues still hold messages after threshold compaction (#1312; `b050c582a`).
- `5c61d6bc9` 2026-03-04 queue during branch summarization (#1803).
- `3166dae72` 2026-04-13 flush queued messages after tree navigation (#3091).
- `35a0d5d62` 2026-07-17 preserve steering vs follow-up semantics through compaction (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:4047`) (#6730).
- `8eda4f5b2` 2026-07-31 reject prompts during manual compaction (`agent-session.ts:1985-1989`); `3852cb2b8` 2026-08-05 send prompts queued during compaction; 0.84.0 messages queued during manual /compact.
- `8e77f8797` 2026-05-28 drain follow-ups queued by `agent_end` handlers (#5115).

**Lesson** — Any phase that occupies the session must own an input queue and a guaranteed drain that preserves delivery semantics.

Related: [[steering-queue]] · [[follow-up-queue]] · [[auto-compaction]] · [[branch-summary]] · [[queued-messages-stranded-at-run-end]] · [[pi--steering-queue|pi]]
