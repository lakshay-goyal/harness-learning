---
type: failure
concepts: [cross-session-memory]
harnesses: [codex]
---
**Symptom** — The model answered from old memory facts as if they were verified now.

**Root cause** — Injected memory carried no staleness signal and the read prompt had no verification policy.

**Fix · [[codex]]** — `5f7c38baa9` 2026-02-28 "Tune memory read-path for stale facts (#13088)": "encode the risk-of-drift vs verification-effort decision rule directly in the read-path prompt" + "Do not present unverified memory-derived facts as confirmed-current." (`codex-rs/ext/memories/templates/memories/read_path.md:50-73`); consolidation v2 "do not restore corrected or deleted claims from older summaries".

**Lesson** — Injected memory needs an explicit staleness/verification policy in the read prompt.

Related: [[cross-session-memory]] · [[memory-overgeneralizes-preferences]] · [[codex--cross-session-memory|codex]]
