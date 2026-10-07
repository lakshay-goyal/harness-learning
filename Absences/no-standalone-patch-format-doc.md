---
type: absence
harnesses: [codex]
---
# no-standalone-patch-format-doc

The 75-line apply_patch prose instructions were deleted; patch-format knowledge comes from the Lark grammar plus the per-model system prompt.

**What's missing**
- No standalone patch-format doc in the tool definition.

**Evidence of decision**
- `8d637ae398` 2026-08-13 deleted the instructions as "unused".

**Implication**
- Providers without custom-tool grammars get no patch spec from the tool ([[patch-envelope-edit]], [[constrained-tool-sampling]]).

Related: [[patch-envelope-edit]] · [[constrained-tool-sampling]] · [[tool-description-design]] · [[Absences]]
