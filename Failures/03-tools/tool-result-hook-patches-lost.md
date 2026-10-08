---
type: failure
concepts: [tool-result-rewriting, extension-event-hooks]
harnesses: [pi]
---
**Symptom** — With several extensions handling `tool_result`, only the last handler's patch survived; earlier redactions/augmentations were silently lost.

**Root cause** — Handlers each returned a patch against the original result; the runner kept the last one (last-writer-wins).

**Fix · [[pi]]** — `2668326e0` 2026-02-06 (#1280): chain `tool_result` patches — each handler sees the result as patched by earlier handlers (`packages/coding-agent/src/core/extensions/runner.ts:1183-1240`).

**Fix · [[pi]]** (same behavior, other hook) — `4e919868f` 2026-04-22 (#3539): `before_agent_start` handlers now chain `systemPromptOptions`, so `ctx.getSystemPrompt()` sees earlier handlers' changes instead of each handler rebuilding from the original (`packages/coding-agent/src/core/extensions/runner.ts:1420-1472`). Other chained events: `message_end` replacement (`40c6eabb8`, `runner.ts:1144-1181`), `before_provider_request` (`runner.ts:1361-1390`); `cache_warming_decision` stays last-wins (`runner.ts:1121-1141`). See [[pi--extension-event-hooks]].

**Lesson** — Transform hooks must compose (fold over handlers), not override each other.

Related: [[tool-result-rewriting]] · [[extension-event-hooks]] · [[side-door-input-bypasses-hooks]] · [[pi--tool-result-rewriting|pi]]
