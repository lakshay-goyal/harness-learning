---
type: absence
harnesses: [codex]
---
# no-per-file-context-labels

AGENTS.md files from different directories are concatenated with blank lines and no path header; truncation is not signaled to the model.

**What's missing**
- `codex-rs/core/src/agents_md.rs:386-419`.

**Evidence of decision**
- Observed design (M5a); the system prompt explains scoping semantics instead of labelling each file.

**Implication**
- The model cannot tell which directory a rule came from or that content was cut; contrast pi's per-file labels in [[context-file-hierarchy]].

Related: [[context-file-hierarchy]] · [[xml-prompt-boundaries]] · [[Absences]]
