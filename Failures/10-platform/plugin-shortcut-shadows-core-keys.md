---
type: failure
concepts: [extension-ui-primitives]
harnesses: [pi]
---
**Symptom** — Extension shortcuts silently overrode core keys (submit, interrupt, exit), making pi unusable or unstoppable with a plugin installed.

**Root cause** — Shortcut registration had no notion of reserved actions; last registration won.

**Fix · [[pi]]** — `54c33f2ad` 2026-01-18: reserved app keys (`app.interrupt`, `app.exit`, `tui.input.submit`, …) cannot be overridden; extensions win over non-reserved built-ins with a startup diagnostic (`packages/coding-agent/src/core/extensions/runner.ts:96-138,673-716`; `packages/coding-agent/CHANGELOG.md:2319`).

**Lesson** — Keep a reserved set of safety-critical bindings (abort, exit) that plugins can never capture; report conflicts visibly.

Related: [[extension-ui-primitives]] · [[abort-propagation]] · [[pi--extension-ui-primitives|pi]]
