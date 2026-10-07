---
type: failure
concepts: [run-settlement, abort-propagation]
harnesses: [opencode]
---
**Symptom** — After an abort or error the session stayed "Working…" / "generating…", or new prompts were refused as busy, because the persisted assistant message had no `time.completed` and no error; later turns also saw an open message.

**Fix · [[opencode]]**
- `796bc390db` 2025-08-13 session stuck in "Working..."; `c398485213` 2025-09-29 TUI stuck "generating" when done.
- `6c9b2c37a5` 2026-02-01 new sessions allowed after errors (stuck busy status).
- `e76cf967e6` 2026-05-14 "finalize interrupted assistant messages" (#27254): on fiber interruption the loop sets `AbortedError` + `time.completed` even when the processor never started (`finalizeInterruptedAssistant`, `packages/opencode/src/session/prompt.ts:1329-1332`).
- `49593c1ec4` 2026-06-21 v2: an active assistant at interruption gets a failed step "Provider turn interrupted" (`packages/core/src/session/runner/llm.ts:311-319`).

**Lesson** — Every exit path, including interruption before the stream consumer starts, must write a terminal state onto the persisted record.

Related: [[run-settlement]] · [[abort-propagation]] · [[terminal-event-required]] · [[opencode--run-settlement|opencode]] · [[opencode--abort-propagation|opencode]]
