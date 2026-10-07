---
type: absence
harnesses: [codex]
---
# no-partial-history-fork

Sub-agent forks are all-or-nothing: "fork last N turns" was removed; only `none` and `all` remain and legacy integers silently mean all.

**What's missing**
- `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:279-302`.

**Evidence of decision**
- `6221a217e2` 2026-10-06 "Remove partial-history subagent forks (#51329)"; default had flipped to all on 2026-04-21 `15b8cde2a4` (#18873). Last-N truncation also had bugs ([[fork-carries-startup-context]]).

**Implication**
- Cache-prefix reuse and simplicity beat context minimization; a fresh child (`none`) is the only small-context option ([[session-fork]], [[codex--session-fork|codex]]).

Related: [[session-fork]] · [[in-process-subagent-threads]] · [[fork-carries-startup-context]] · [[cache-stable-prompt-prefix]] · [[Absences]]
