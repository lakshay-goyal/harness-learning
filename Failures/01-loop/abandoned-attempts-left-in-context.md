---
type: failure
concepts: [auto-retry-backoff, context-edit-overlay, context-projection]
harnesses: [pi, opencode]
---
**Symptom** — Failed attempts from selected error retries and from final length/overflow recovery remained in future provider context.

**Root cause** — Raw transcript (audit log) and provider context were the same thing; agent in-memory state, not the log, defined the request.

**Fix · [[pi]]** — `466db0fec` 2026-09-21 "add canonical session context boundaries" (`packages/coding-agent/CHANGELOG.md:368`, 0.87.0): SessionManager projections authoritative for provider requests via `prepareRequest` (`packages/coding-agent/src/core/agent-session.ts:787-844`); `_prepareRetry` and overflow recovery append a `context_edit` null omission for the failed attempt while keeping raw history (`agent-session.ts:3785-3786`, `_omitRecoveryAttempt` `:1236-1252`); "preserve existing queue scheduling during continuation and recovery" (`packages/agent/src/agent-loop.ts:173-285`).

**Fix · [[opencode]]** partial. A retried attempt reuses the same assistant message and resets only in-memory cursors, so parts persisted by the failed attempt stay in SQLite (`packages/opencode/src/session/processor.ts:649-652`). `cleanup()` marks its tool calls `metadata.interrupted`, and `748fcb7ebd` 2026-05-25 stopped those orphans from re-entering the loop as pending work (`packages/opencode/src/session/prompt.ts:96-100`). Partial text from the failed attempt is still replayed (inference, unfixed).

**Lesson** — Keep the raw transcript and the provider projection separate; retries and recovery edit the projection only.

Related: [[auto-retry-backoff]] · [[context-edit-overlay]] · [[context-projection]] · [[failed-turns-replayed]] · [[pi--auto-retry-backoff|pi]] · [[opencode--auto-retry-backoff|opencode]]
