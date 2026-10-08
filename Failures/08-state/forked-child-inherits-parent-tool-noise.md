---
type: failure
concepts: [session-fork, in-process-subagent-threads]
harnesses: [codex]
---
**Symptom** — Full-history sub-agent forks copied the parent's reasoning and tool calls (including the unmatched in-flight `spawn_agent` call), and parent-specific developer instructions (role hints, usage hints, multi-agent mode text, Guardian denial notes) leaked into the child, which then behaved as the parent's role.

**Root cause** — Fork reused the user-fork primitive (copy prefix) for a different purpose: the child needs the parent's *conversation*, not its *identity*, tool scaffolding or private reasoning.

**Fix · [[codex]]**
- `567d2603b8` 2026-04-03 "Sanitize forked child history" — keep only system/developer/user messages + assistant PartialAnswer/FinalAnswer; drop reasoning, tool calls/outputs, web-search/image calls, compaction triggers, `TokenUsageRecord`; "remove the unmatched synthetic spawn output" (`codex-rs/core/src/agent/control/spawn.rs:88-132`).
- `663da53823` 2026-08-20 — per-content-item filtering of parent developer content (role instructions, usage hints, multi-agent mode, current-time reminders, Guardian denial prefix) before the child's own instructions are appended (`codex-rs/core/src/agent/control/spawn.rs:134-165`).
- `6221a217e2` 2026-10-06 — strip stale hints "by kind, including persisted hints whose wording predates the current bundled instructions".

**Lesson** — A forked child needs the parent's conversation, not its agent role; strip identity/role/tool scaffolding by structural tag (content kind), not by string match.

Related: [[session-fork]] · [[in-process-subagent-threads]] · [[xml-prompt-boundaries]] · [[codex--session-fork|codex]] · [[codex--in-process-subagent-threads|codex]] · [[fork-carries-startup-context]]
