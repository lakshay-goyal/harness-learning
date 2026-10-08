---
type: failure
concepts: [tool-output-truncation, cross-provider-handoff]
harnesses: [codex]
---
**Symptom** — "Replaying tool outputs under a different model can expand or shrink the history shown to the model if truncation uses the new model's budget." (`aa88a0333c` body) — resume, fork or model switch changed what the model had seen.

**Root cause** — Truncation is applied at history-record time from the *current* model's policy, while the rollout keeps full payloads; replay re-truncated with whichever model was active.

**Fix · [[codex]]** — `aa88a0333c` 2026-09-09 "Preserve tool output truncation budgets across resume and fork": persist the originating budget on each tool output (`history_truncation_token_limit` metadata), used as-is on record/replay (`codex-rs/core/src/context_manager/history.rs:566-574`; `replay_annotated_item` `:531-550`). Items without the metadata still fall back to the current policy.

**Lesson** — Store the truncation budget with the result so resume/fork/model-switch replays are deterministic.

Related: [[tool-output-truncation]] · [[cross-provider-handoff]] · [[context-projection]] · [[codex--tool-output-truncation|codex]]
