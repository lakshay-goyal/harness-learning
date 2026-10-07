---
type: failure
concepts: [auto-compaction, run-settlement]
harnesses: [codex]
---
**Symptom** — Pre-turn compaction runs before the incoming input is recorded; when it failed, the accepted user prompt was never written to history, and the error was reported before prompt hooks ran.

**Root cause** — Order "compact → record input" plus early-return on compaction failure.

**Fix · [[codex]]** — `ee93abb690` 2026-09-10 "Preserve incoming prompts when pre-turn compaction fails (#44487)": record input and run prompt hooks on every pre-turn compaction failure; defer error events — "Pre-turn failures are reported after preserving the incoming prompt" (`codex-rs/core/src/compact.rs:250`). Related ordering fixes: `26590d7927` 2026-01-28 compaction starts after `TurnStarted`; `bb0ac5be70` 2026-02-20 context recorded before pre-turn compaction re-injected → [[compaction-cancellation-races]].

**Lesson** — Never let a pre-step maintenance failure lose user input already accepted; decide the order "record input → compact → re-inject context → sample" explicitly.

Related: [[auto-compaction]] · [[run-settlement]] · [[side-phase-input-lost]] · [[compaction-cancellation-races]] · [[codex--auto-compaction|codex]]
