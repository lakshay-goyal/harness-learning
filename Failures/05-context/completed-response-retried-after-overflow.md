---
type: failure
concepts: [overflow-recovery]
harnesses: [pi, opencode]
---
**Symptom** — A response that finished normally but whose usage showed input > context window (silent overflow) was compacted and then "retried", re-running an already completed answer.

**Root cause** — The overflow path always set `willRetry`; it didn't distinguish an error/length stop (nothing usable produced) from a successful `stop`. `agent.continue()` also cannot continue from a completed assistant message.

**Fix · [[pi]]** — `6b9f3f492` 2026-06-18 (#5720): `willRetry = stopReason !== "stop"`; case 2 compacts without retry (`packages/coding-agent/src/core/agent-session.ts:3010-3016` at HEAD).

**Fix · [[opencode]]** Prevented by design in v2 `820c984d47` 2026-06-05 (#31005): overflow recovery only when `!publisher.hasAssistantStarted()` — an overflow after durable assistant output is an ordinary terminal failure (`packages/core/src/session/runner/llm.ts:246`, `:291-297`; `specs/v2/session.md:121`). Legacy marks `overflow: !handle.message.finish` so a finished response compacts without replaying the user turn (`packages/opencode/src/session/prompt.ts:1320-1328`).

**Lesson** — Recovery must not re-run a turn that already succeeded; compact for the *next* request instead.

Related: [[overflow-recovery]] · [[context-overflow-detection]] · [[overflow-compaction-cascade]]
