---
type: failure
concepts: [auto-compaction, extension-event-hooks]
harnesses: [pi]
---
**Symptom** — When auto-compaction failed (e.g. quota exceeded on the summary call) the error was thrown and crashed the interactive UI; later, extensions had no signal that compaction failed.

**Root cause** — Background compaction errors propagated as exceptions out of an event handler instead of being reported as lifecycle events.

**Fix · [[pi]]**
- `20f5fcc79` 2026-01-16 (#792): "emit the error via the auto_compaction_end event instead of throwing. The UI now displays the error message, allowing users to take action (switch models, wait for quota reset, etc.)". HEAD: `compaction_end{errorMessage: "Auto-compaction failed: …" | "Context overflow recovery failed: …"}` (`packages/coding-agent/src/core/agent-session.ts:3211-3236`).
- `a6b1dbceb` 2026-08-17 (#8241) `session_compact_failed{reason, errorMessage?, aborted, willRetry, fromExtension}` for extensions (`packages/coding-agent/src/core/extensions/types.ts:794-800`).

**Lesson** — A failed background summary must leave the session alive and history untouched, surfaced as an event the UI and plugins can act on.

Related: [[auto-compaction]] · [[summary-call-not-retried]] · [[errors-as-stream-events]]
