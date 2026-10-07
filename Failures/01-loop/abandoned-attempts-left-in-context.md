---
type: failure
concepts: [auto-retry-backoff, context-edit-overlay, context-projection]
harnesses: [pi]
---
**Symptom** — Failed attempts from selected error retries and from final length/overflow recovery remained in future provider context.

**Root cause** — Raw transcript (audit log) and provider context were the same thing; agent in-memory state, not the log, defined the request.

**Fix · [[pi]]** — `466db0fec` 2026-09-21 "add canonical session context boundaries" (`packages/coding-agent/CHANGELOG.md:368`, 0.87.0): SessionManager projections authoritative for provider requests via `prepareRequest` (`packages/coding-agent/src/core/agent-session.ts:787-844`); `_prepareRetry` and overflow recovery append a `context_edit` null omission for the failed attempt while keeping raw history (`agent-session.ts:3785-3786`, `_omitRecoveryAttempt` `:1236-1252`); "preserve existing queue scheduling during continuation and recovery" (`packages/agent/src/agent-loop.ts:173-285`).

**Lesson** — Keep the raw transcript and the provider projection separate; retries and recovery edit the projection only.

Related: [[auto-retry-backoff]] · [[context-edit-overlay]] · [[context-projection]] · [[failed-turns-replayed]] · [[pi--auto-retry-backoff|pi]]
