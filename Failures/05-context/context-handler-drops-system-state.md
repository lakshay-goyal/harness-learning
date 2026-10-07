---
type: failure
concepts: [context-transform-hook, transcript-carried-system-prompt, message-conversion-layer]
harnesses: [pi]
---
**Symptom** — After extension-driven compaction, requests went out without built-in tools and Codex emitted raw tool-call text instead of calling tools.

**Root cause** — Since `9e05370b2` (#9548, 2026-09-16) the system prompt and tool declarations live in the transcript as system messages. Extension `context` handlers that sliced/windowed the message list (e.g. "keep from the compaction summary on") dropped the leading system message, deleting prompt and tools from the request.

**Fix · [[pi]]** — `aef5fc429` 2026-09-21 (#9846) "keep prompt and tool state across context handlers": `context` handlers now see only non-system messages and Pi restores the replayed system message afterwards (`restoreSystemMessages`, `packages/coding-agent/src/core/extensions/runner.ts:276-293`, `1298-1328`); new `context_with_system` event for handlers that really want the full transcript (Pi reports, but honors, a removed leading system message) (`packages/coding-agent/src/core/extensions/types.ts:866-884`).

**Lesson** — If tools/prompt live in the message list, guard them from user-level message transforms: give plugins the conversation, keep harness state out of their reach.

Related: [[context-transform-hook]] · [[transcript-carried-system-prompt]] · [[message-conversion-layer]] · [[extension-event-hooks]]
