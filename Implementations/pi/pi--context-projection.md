---
type: implementation
harness: pi
concept: context-projection
commit: b30a6dd77
files: [packages/coding-agent/src/core/agent-session.ts:787, packages/coding-agent/src/core/session-manager.ts:390, packages/coding-agent/src/core/session-manager.ts:476, packages/coding-agent/src/core/session-manager.ts:543, packages/durable/src/harness/context.ts:9, packages/durable/docs/spec.md:267]
---
[[context-projection]] in [[pi]].

## Mechanism
- **Authoritative projection**: AgentSession installs `prepareRequest` (runs before *every* provider request, [[turn-lifecycle-hooks]]) which replaces `context.messages` with `sessionManager.buildSessionProjection().messages` (`packages/coding-agent/src/core/agent-session.ts:787-844`, esp. `:793-799`); agent state refreshed from it after finalization (`_refreshFinalizedContext`, `:938-944`). Introduced `466db0fec` "Make SessionManager projections authoritative for provider requests".
- **Rebuild pipeline** (`packages/coding-agent/src/core/session-manager.ts:390-583`):
  1. `buildSessionPath`: walk leaf→root (`:390-416`; linear since `a1da88aed`).
  2. Model/thinking from `model_change`/`thinking_level_change` and last assistant on path (`:418-433`).
  3. `buildContextEntries` (`:476-512`): if compactions exist on the path use the **latest**: `[compaction]` + non-system path entries from `firstKeptEntryId` up to the compaction + all entries after.
  4. Compaction projects to `[systemMessage checkpoint, compactionSummary]` (`:461-464`); older retained compactions project nothing (`:555-565`).
  5. `context_edit` latest-per-target applied: `null` omits, content replace keeps role/metadata (`:519-540,551-554`) → [[context-edit-overlay]].
  6. Null content normalized at load (`:439-452`; `8c0ccd14b`).
- **Then per request** (agent-core): `transformContext` chain (extension `context` event → hidden-declarations projection → forced-system-prompt projection, `agent-session.ts:1766-1801`) → `convertToLlm` → pi-ai `transformMessages` provider repair ([[transcript-replay-repair]]).
- **System prompt is part of projection**: transcript-carried `SystemMessage` deltas replayed; compaction stores a full `systemMessage` checkpoint (`session-manager.ts:1261-1287`) → [[transcript-carried-system-prompt]].
- **Compaction trigger uses projection**: `_compactBeforeNextAssistantResponse` builds the projection and checks threshold (`agent-session.ts:776-785`); `estimateProjectedContextTokens` trusts usage only if it is newer than the latest `context_edit`/compaction, else estimates (`packages/coding-agent/src/core/compaction/compaction.ts:226-262`) → [[token-estimation]].
- **Branch-relative**: only the active path is projected; abandoned branches never enter context (compaction too, `ddda8b124`).

## Constants
- none (pure derivation); see [[auto-compaction]] for reserve/keep tokens.

## Evolution
- 2025-12-04 `79731249e` compaction entries + rebuild (initial).
- 2025-12-25 `c58d5f20a` tree path walk.
- 2025-12-31 `ddda8b124` compaction uses current branch path only.
- 2026-06-20 `a1da88aed` linear path traversal (#5909).
- 2026-09-21 `466db0fec` canonical session context boundaries: projection authoritative for every request; `context_edit`; failed attempts omitted (CHANGELOG 0.87.0, `packages/coding-agent/CHANGELOG.md:368`).
- 2026-09-28 `540e174c7` virtual models: per-request threshold vs routed physical model inside `prepareRequest` (`agent-session.ts:836-841`).

## Evidence commits
`79731249e`, `c58d5f20a`, `ddda8b124`, `a1da88aed`, `466db0fec`, `540e174c7`

## Quirks
- Two context paths coexist: agent-core `Agent` keeps its own `state.messages` (SDK users without SessionManager), coding-agent overrides per request.
- Leaf not persisted ⇒ projection after resume follows the last-appended entry, not the last navigated leaf (inferred).

## Durable variant (packages/durable)
- Context derived per request from immutable entries (`packages/durable/docs/spec.md:267-290`; `src/harness/context.ts`): active range starts at the newest **head marker** (`pi.reset` or `pi.compaction`); edits `omit`/`replace`; tool results moved to directly follow their call; missing results synthesized "Tool result unavailable: history ends before this call completed." (`context.ts:10`); assistants with `aborted`/`error`/`deferred` excluded (`context.ts:9`); a system message preceded only by user messages moved to front (`context.ts:171-176`; `92216fa15` cache fix). After a head cut the next system entry is a complete baseline (`spec.md:3223-3234`). History never deleted (`spec.md:3320-3321`). `context({at})` reads historical context (`76f6c06da`).
- Caching: `68ccef176` reuse scanned range within a task invocation; `da866ada1` keep each conversation's last range in memory for `contextRetentionMs` (default 600 000, `packages/durable/src/harness/agent.ts:52-53,71`); `ae92585d3` throwing settings getters must not break the retention check → [[per-request-projection-rescans-log]].

## Failures
- [[abandoned-attempts-left-in-context]] · [[per-request-projection-rescans-log]] · [[quadratic-long-session-operations]]
