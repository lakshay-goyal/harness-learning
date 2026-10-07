---
type: absence
harnesses: [codex]
---
# no-tool-output-spill-file

Truncated tool output is not written to a model-readable temp file; the full output survives only in the rollout JSONL, which the model is not pointed at.

**What's missing**
- `codex-rs/core/src/context_manager/history.rs:514-515`.

**Evidence of decision**
- Observed design (M6). In token-budget sessions the model instead gets server-side history/notes search tools (`daa48072f4` 2026-08-21) → [[model-requested-context-reset]].

**Implication**
- The model must re-run a command to see elided output; contrast pi's spill file ([[tool-output-spill]], [[tool-output-truncation]]). Rollout keeps full function-call payloads → [[unbounded-payload-in-transcript]].

Related: [[tool-output-truncation]] · [[tool-output-spill]] · [[model-requested-context-reset]] · [[unbounded-payload-in-transcript]] · [[Absences]]
