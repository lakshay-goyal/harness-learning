---
type: failure
concepts: [steering-queue, follow-up-queue, auto-compaction, abort-propagation]
harnesses: [pi, codex]
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

**Fix · [[codex]]**
- Symptom: steered input during/after mid-turn compaction was drained before the model had answered the active prompt → `8f705b0702` 2026-04-09 "Defer steering until after sampling the model post-compaction (#17163)" (`can_drain_pending_input`, `codex-rs/core/src/session/turn.rs:604-611`).
- Symptom: interrupting while MCP servers were still starting aborted before the submitted input was recorded (`d7e8f4c3dc` 2026-07-22 "Preserve user input when MCP startup is interrupted (#34839)" — MCP tool list and router built per step snapshot under the turn cancellation token); earlier Esc interrupt broken when MCP startup completed mid-turn (`3c711f3d16` 2026-01-13).
- Input recorded even when pre-sampling compaction fails (`codex-rs/core/src/session/turn.rs:193-201`) or prewarm is cancelled (`codex-rs/core/src/tasks/regular.rs:82-91`). TUI queues follow-ups during manual `/compact` (`e838645fa2`).

**Lesson** — Any phase that occupies the session must own an input queue and a guaranteed drain that preserves delivery semantics; record accepted user input before any cancellable preparation.

Related: [[steering-queue]] · [[follow-up-queue]] · [[auto-compaction]] · [[branch-summary]] · [[queued-messages-stranded-at-run-end]] · [[pi--steering-queue|pi]] · [[mcp-integration]] · [[codex--steering-queue|codex]]
