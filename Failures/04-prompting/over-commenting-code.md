---
type: failure
concepts: [per-model-system-prompt, guideline-softening]
harnesses: [opencode]
---
**Symptom** — Models added code comments a human in that codebase would not write (narration, chain-of-thought in comments).

**Root cause** — Model habit; per-family prompts disagree on whether to forbid it.

**Fix · [[opencode]]**
- default.txt "IMPORTANT: DO NOT ADD ***ANY*** COMMENTS unless asked" (`packages/opencode/src/session/prompt/default.txt:68`); removed from anthropic.txt in `795b845782` 2025-10-25.
- meta.txt "NEVER use comments as a place for long-winded chain-of-thought" (`5a8ee27254` 2026-07-20).
- Team cleanup command `.opencode/command/rmslop.md` ("remove all AI generated slop": extra comments, defensive try/catch, `any` casts, emoji).
- origin/v2 `9dd7149e75` 2026-09-11: the one Claude-specific line kept after the Anthropic prompt was removed — "By default, match the surrounding comment density: where the code has none, add none." (`origin/v2:packages/core/src/plugin/system-prompt/anthropic.txt:1-2`).

**Lesson** — Anchor comment rules to the surrounding code's density rather than an absolute ban.

Related: [[per-model-system-prompt]] · [[guideline-softening]] · [[imperative-guideline-over-compliance]] · [[opencode--guideline-softening|opencode]]
