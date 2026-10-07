---
type: implementation
harness: pi
concept: context-edit-overlay
commit: b30a6dd77
files: [packages/coding-agent/src/core/session-manager.ts:173, packages/coding-agent/src/core/session-manager.ts:1360, packages/coding-agent/src/core/session-manager.ts:519, packages/coding-agent/src/core/agent-session.ts:1236, packages/coding-agent/src/core/agent-session.ts:3786, packages/coding-agent/docs/session-format.md:139, packages/coding-agent/docs/compaction.md:85]
---
[[context-edit-overlay]] in [[pi]].

## Mechanism
- **Entry** `ContextEditEntry {type:"context_edit", targetId, replacement: {content} | null}` — "Append-only change to one earlier entry's contribution to model context"; `null` omits, a value "replaces only its content" (`packages/coding-agent/src/core/session-manager.ts:166-180`).
- **Doc contract** (`packages/coding-agent/docs/session-format.md:139-147`): changes only future model context; raw history, UI, exports and session accounting unchanged; targets user / assistant / toolResult / custom_message entries; string replacement for assistant/toolResult normalized to one text block; **latest edit on the active branch wins**; **branch-relative** — navigating before the edit reveals the original.
- **`appendContextEdit(targetId, replacement)`** (`session-manager.ts:1360-1398`): validates replacement shape; target must exist, be **on the active branch** (`getBranch()`), and be an editable role; appended as a normal child of the leaf (so it moves the leaf).
- **Application**: `buildSessionProjection` applies latest-per-target edits (`session-manager.ts:519-540,551-554`) → [[context-projection]].
- **Harness writers**:
  - Retry: `_prepareRetry` omits the failed assistant via `_omitRecoveryAttempt(message)` before backoff (`packages/coding-agent/src/core/agent-session.ts:3761-3801`, `:3786`) → [[auto-retry-backoff]].
  - Overflow recovery: `_checkCompaction` case 1 omits the failed attempt + its tool results, then compacts and retries once (`agent-session.ts:3042`) → [[overflow-recovery]].
  - `_omitRecoveryAttempt` (`agent-session.ts:1236-1252`): resolves each target to its persisted entry id; throws "Cannot persist recovery omission because a projected message has no source entry" if a projected message has no entry; appends `context_edit … null` per target, emits `entry_appended`, refreshes finalized context.
  - Documented ordering: persist final assistant → `turn_end` → `agent_end` → append omissions → `session_before_compact` + compaction → retry as fresh run; on failure keep omissions, no compaction (`packages/coding-agent/docs/compaction.md:85-98`).
- **Extension writers**: `turn_end` / `agent_before_settle` handlers may append `context_edit` (and compaction) drafts at boundaries (`agent-session.ts:946-985`; `packages/coding-agent/src/core/extensions/types.ts:944-977`).
- **Token accounting**: if the projection contains edits, `estimateProjectedContextTokens` trusts provider usage only when recorded after the latest edit/compaction (`packages/coding-agent/src/core/compaction/compaction.ts:226-262`; `agent-session.ts:3056-3057`) → [[token-estimation]].

## Constants
- none.

## Evolution
- Before 2026-09-21: failed retry attempts and final length/overflow recovery attempts stayed in future provider context ([[abandoned-attempts-left-in-context]]); provider-boundary `transformMessages` only skipped `error`/`aborted` assistants (`fbb74bb29`, 2026-01-16).
- 2026-09-21 `466db0fec` "add canonical session context boundaries": `context_edit` entry, `_omitRecoveryAttempt`, projections authoritative (CHANGELOG 0.87.0, `packages/coding-agent/CHANGELOG.md:368`).

## Evidence commits
`466db0fec`, `fbb74bb29`

## Quirks
- Each edit moves the leaf (it is a tree node), so edits interleave with conversation entries on the path.
- Edits target entry ids; forks copy the path including edits (same ids retained by `createBranchedSession`).

## Durable variant (packages/durable)
- Edits `omit`/`replace` target entries in the derived context (`packages/durable/docs/spec.md:267-290`); applied relative to the newest head marker.

## Failures
- [[abandoned-attempts-left-in-context]]
