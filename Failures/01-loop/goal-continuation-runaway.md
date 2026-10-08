---
type: failure
concepts: [persistent-goal-continuation]
harnesses: [codex]
---
**Symptom** — Automatic `/goal` turns kept firing when no progress was possible: permanent errors (e.g. compaction HTTP 400), hard usage exhaustion replaying failing turns, repeated permission/external blockers, empty answers, failing exec hosts — "`/goal` can keep synthesizing turns even when the next turn cannot make meaningful progress… keep burning tokens" (`0d344aca9b` body; issues #22833, #22245, #23067).

**Root cause** — The idle-continuation driver relied on the model to declare completion/blockage; no harness-side terminal states for "blocked" or "out of quota", no repetition threshold.

**Fix · [[codex]]**
- `0d344aca9b` 2026-05-18 "goal: pause continuation loops on usage limits and blockers (#23094)" — resumable `blocked` / `usageLimited` states + model-side Blocked audit (same blocker ≥ 3 consecutive goal turns, `codex-rs/ext/goal/templates/goals/continuation.md:49-54`).
- `c62d79259d` 2026-06-05 — any terminal turn error → Blocked.
- `0cdb1f1c83` 2026-08-25 — prompt No-progress check + equivalent-blocker rule.
- `62b458c931` 2026-08-29 — Blocked after 3 consecutive exec-failure turns (`codex-rs/ext/goal/src/accounting.rs:155-160`).
- `0735c51978` 2026-09-09 — Blocked after 3 consecutive empty automatic turns (`codex-rs/ext/goal/src/accounting.rs:216-230`).

**Lesson** — An autonomous continuation loop needs explicit terminal states (blocked, usage-limited) and harness-side repetition breakers, not only model self-reporting; mirror the harness threshold in the model-facing rule.

Related: [[persistent-goal-continuation]] · [[unbounded-hook-continuation-loop]] · [[goal-loop-premature-stop]] · [[session-token-budget]] · [[no-turn-cap]] · [[codex--persistent-goal-continuation|codex]]
