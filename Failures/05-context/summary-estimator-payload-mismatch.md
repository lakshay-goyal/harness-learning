---
type: failure
concepts: [token-estimation, auto-compaction]
harnesses: [codex]
---
**Symptom** — Remote compaction's pre-trim estimate used model-derived base instructions while the actual compaction payload used the session's instructions — deterministic over/under trimming.

**Root cause** — Estimator and request builder computed the prompt independently.

**Fix · [[codex]]** — `dc7007beaa` 2026-02-04 (#10692): estimation and request share `base_instructions` (`codex-rs/core/src/compact_remote_v2_attempt.rs:27-41`). Same family as the bytes-as-tokens reasoning bug (`1fc72c647f`) → [[estimator-undercounts-context]].

**Lesson** — Estimate exactly the bytes you will send: build the payload once and measure that.

Related: [[token-estimation]] · [[auto-compaction]] · [[estimator-undercounts-context]] · [[codex--token-estimation|codex]]
