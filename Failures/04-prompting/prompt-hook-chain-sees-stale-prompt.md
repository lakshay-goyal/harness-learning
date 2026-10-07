---
type: failure
concepts: [system-prompt-override, extension-event-hooks]
harnesses: [pi]
---
**Symptom** — With several extensions on `before_agent_start`, `ctx.getSystemPrompt()` inside a later handler returned the base prompt, ignoring earlier handlers' changes — handlers overwrote each other or built on stale text (#3539).

**Root cause** — Each handler received the original prompt; results were not chained through the handler list.

**Fix · [[pi]]** — `4e919868f` 2026-04-22 (#3539) "chain system prompt in before_agent_start". HEAD: one mutable `NormalizedBuildSystemPromptOptions` shared across handlers ("Later handlers observe mutations made by earlier handlers", `packages/coding-agent/src/core/extensions/types.ts:927-928`); `event.systemPrompt` and `ctx.getSystemPrompt()` re-render from current options; a returned `systemPrompt` sets `forceSystemPrompt` that later handlers observe (`extensions/runner.ts:1420-1472`).

**Lesson** — Transform hooks must compose: pass each handler the accumulated state, not the original.

Related: [[system-prompt-override]] · [[extension-event-hooks]] · [[tool-result-hook-patches-lost]] · [[pi--system-prompt-override|pi]]
