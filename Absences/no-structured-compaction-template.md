---
type: absence
harnesses: [codex]
---
# no-structured-compaction-template

The local summarizer prompt is 9 free-form lines with no fixed sections.

**What's missing**
- `codex-rs/prompts/templates/compact/prompt.md:1-9`; summary prefixed by a short handoff framing (`codex-rs/prompts/templates/compact/summary_prefix.md`).

**Evidence of decision**
- Original structured template (Objective / User instructions / AI actions / Important entities / Open issues, `6cfee15612`) dropped 2025-09-12 `ea225df22e`; a strict-JSON structured variant was tried on a side branch and not merged (`e39e0c4332`, rejected experiment).

**Implication**
- Preservation delegated to code (kept user messages verbatim, 20_000-token budget) and, for OpenAI models, to opaque server-side compaction; contrast pi's [[structured-compaction-summary]] ([[compaction-design]]).

Related: [[structured-compaction-summary]] · [[auto-compaction]] · [[compaction-design]] · [[no-summary-validation-local]] · [[Absences]]
