---
type: failure
concepts: [extension-event-hooks]
harnesses: [pi]
---
**Symptom** — Hook handlers that legitimately waited (an LLM call inside a hook, a confirmation dialog waiting on the human) were killed by the hook timeout, so plugins failed mid-action.

**Root cause** — The original hooks system (`04d59f31e`, 2025-12-09) imposed a `hookTimeout` on every handler; it was applied inconsistently and cannot distinguish "stuck" from "waiting on a human/model".

**Fix · [[pi]]** — `88e39471e` 2025-12-31 "Remove hook execution timeouts": no wall-clock limit; user aborts with Ctrl+C/Esc instead. Corollary: `675756154` 2026-07-02 tried to make stuck `context` hooks abortable (#6234) and was reverted 5 days later by `2b00dade7` 2026-07-07 (reason not stated — unverified). At HEAD handlers are fully sequential and awaited, so a stuck hook still stalls the run (`packages/coding-agent/docs/extensions.md:109-111`).

**Lesson** — Don't put wall-clock timeouts on plugin code that may await humans or models; give the user a reliable abort and make the hang visible instead.

Related: [[extension-event-hooks]] · [[abort-propagation]] · [[pi--extension-event-hooks|pi]]
