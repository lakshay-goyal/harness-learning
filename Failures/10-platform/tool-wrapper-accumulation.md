---
type: failure
concepts: [plugin-tools, runtime-plugin-loading]
harnesses: [opencode]
---
**Symptom** — Object-defined tools (plugin or custom tools passed as a definition object) got their `execute` wrapped again on every re-initialization: validation and truncation ran repeatedly and the wrapper stack grew.

**Root cause** — `Tool.define()` mutated the caller-provided definition object in place, so the next init wrapped the already-wrapped function.

**Fix · [[opencode]]** — `81d3ac3bf0` 2026-04-02 "prevent Tool.define() wrapper accumulation on object-defined tools" (#16952): shallow-copy the definition before wrapping.

**Lesson** — Factories must never mutate caller-provided definition objects; wrap a copy so re-init is idempotent.

Related: [[plugin-tools]] · [[runtime-plugin-loading]] · [[plugin-registrations-leak-after-failure-or-reload]] · [[opencode--plugin-tools|opencode]]
