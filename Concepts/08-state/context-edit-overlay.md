---
type: concept
stage: state
tier: candidate
aliases: [context_edit, ContextEditEntry, appendContextEdit, _omitRecoveryAttempt, "replacement: null", omit/replace edits, context-edit-omission]
harnesses: [pi]
---
Append-only log entries that hide (omit) or replace the model-visible content of an earlier entry, applied only when building the request context — raw history, UI, exports and billing keep the original.

## Why
- Retries and overflow recovery must not show the model its own failed/truncated attempt, yet deleting it loses audit trail and breaks the append-only log ([[abandoned-attempts-left-in-context]]).
- Plugins need to prune or rewrite bulky past content (e.g. huge tool output) without mutating history or forking.

## Design space
- **Delete/rewrite entries in place** (breaks append-only, crash-unsafe) vs **edit entries targeting an id** (pi) vs **per-request transform hook only** (non-persistent; pi `context` event).
- **Edit ops**: omit (`replacement: null`) and content replace keeping role/metadata (pi coding-agent; durable `omit`/`replace`).
- **Scope**: branch-relative (edit only applies where it is on the active path) — pi.
- **Validation**: target must be on the active branch and of an editable role (pi `appendContextEdit`).
- **Last-wins per target** (pi) vs stacked.
- **Who writes**: harness recovery paths (retry, overflow) and plugins at turn boundaries (pi drafts at `turn_end`/`agent_before_settle`).
- **Token accounting**: usage captured before an edit is untrusted → estimate (pi `estimateProjectedContextTokens`).
- codex: absent as per-entry omit/replace. Nearest design: append-only `ThreadRolledBack{num_turns}` markers that drop the last N turns at replay (`8b7ec31ba7` 2026-01-06; replayed by `codex-rs/core/src/context_manager/history.rs:755-880`); API removed `3052bbcf8c` 2026-09-11 in favour of `thread/revert` writing a new rollout file ([[codex--session-fork|codex session-fork]]). Failed attempts are not hidden: completed items from a failed attempt stay in history (`codex-rs/core/src/session/turn.rs:3156-3165`).

## Implementations
- [[pi--context-edit-overlay|pi]] — `ContextEditEntry {targetId, replacement:{content}|null}` (`466db0fec`); retry and overflow omit failed attempts; extensions append drafts; latest-per-target in `buildSessionProjection`.

## Failures
- [[abandoned-attempts-left-in-context]]

## Related
[[context-projection]] · [[session-tree]] · [[auto-retry-backoff]] · [[overflow-recovery]] · [[context-transform-hook]] · [[token-estimation]] · [[extension-event-hooks]]
