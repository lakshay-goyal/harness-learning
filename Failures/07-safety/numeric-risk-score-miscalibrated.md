---
type: failure
concepts: [llm-approval-reviewer, structured-classifier-api]
harnesses: [codex]
---
**Symptom** — The reviewer and the async classifier emitted numeric risk scores that needed per-prompt calibration: MVP auto-approved only when `risk_score < 80` (`e84ee33cc0`); Guardian V2 reviewed at `action_risk ≥ 0.5` but custom prompts kept a 0.8 legacy threshold (`846a16852f`; `codex-rs/ext/guardian-v2/src/async_scorer/config.rs:28-29`).

**Root cause** — LLMs produce poorly calibrated numbers; a threshold tuned for one prompt/model breaks with the next.

**Fix · [[codex]]** — 2026-04-08 `dcbc91fd39` sync reviewer → enum `risk_level` + `user_authorization` → `outcome` derived by an explicit rule in the prompt (`codex-rs/prompts/templates/guardian/policy_template.md:66-73`; `codex-rs/ext/guardian-reviewer/src/assessment.rs:80-100`); 2026-04-21 `ef00014a46` bare `{"outcome":"allow"}`; 2026-08-24 `a9e7920da1` async classifier → single first token `high`/`low` mapped to 1.0/0.0, returned as soon as streamed (`codex-rs/prompts/templates/guardian/classifier_instructions.md:82`).

**Lesson** — Ask an LLM judge for categorical labels combined by an explicit rule, not calibrated numbers; a single-token verdict also minimizes latency.

Related: [[llm-approval-reviewer]] · [[structured-classifier-api]] · [[codex--llm-approval-reviewer|codex]] · [[approval-reviewer-overcautious]]
