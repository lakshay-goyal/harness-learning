---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — After the system prompt was restructured into XML sections, the eval's `without_docs` transform silently stopped removing the docs section — its plain-text markers (`\nGuidelines:\n`, `Current working directory:`) no longer existed, so control and treatment arms got the same prompt.

**Root cause** — The treatment was applied by string surgery keyed on prompt text owned by another part of the codebase.

**Fix · [[pi]]** — `1247476e6` 2026-09-16 switched to `<rules>`/`<docs>`/`<cwd>` markers; the transform fails closed when markers are missing (`packages/evals/src/harness.ts:498-502`) and `verifySystemPrompt` errors if `<rules>` is absent or `<docs>` presence ≠ variant (`harness.ts:257-270`).

**Lesson** — Treatment transforms must verify they applied and fail closed when their anchors move.

Related: [[harness-evals]] · [[xml-prompt-boundaries]] · [[self-documentation-pointer]] · [[pi--harness-evals|pi]]
