---
type: failure
concepts: [per-model-system-prompt]
harnesses: [codex]
---
**Symptom** — After the GPT-5 prompt rewrite `81b148bda2` 2025-08-07 (+265/−75 lines) dropped shell guidance, "we're seeing some specific points of confusion from the model!" (`90d892f4fd` body).

**Root cause** — Lines that fixed earlier observed behaviours had no recorded provenance, so a wholesale rewrite deleted them.

**Fix · [[codex]]** — `90d892f4fd` 2025-08-12 "[prompt] Restore important guidance for shell command usage (#2211)": restored `rg` preference ("(If the `rg` command is not found, then use alternatives.)"), chunked reads, and renamed network modes ON/OFF → **restricted**/**enabled** ("anecdotally less confusing to the model and requires less reasoning to escalate for approval"); validated with "[x] evals". (The restored chunk/limit numbers later went stale → [[prompt-states-stale-harness-limits]].)

**Lesson** — Prompt rewrites need per-line provenance (which failure each line fixes) and an eval pass; deleted lines silently reintroduce old failures.

Related: [[per-model-system-prompt]] · [[prompt-states-stale-harness-limits]] · [[harness-evals]] · [[codex--per-model-system-prompt|codex]]
