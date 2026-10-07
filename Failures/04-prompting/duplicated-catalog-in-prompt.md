---
type: failure
concepts: [skill-progressive-disclosure, cache-stable-prompt-prefix]
harnesses: [opencode]
---
**Symptom** — The skill catalog was rendered twice — in the system prompt and in the skill tool description (and a third time in kimi.txt) — costing tokens and making the tool description change whenever skills changed.

**Root cause** — Catalog added to a new channel without removing it from the old ones.

**Fix · [[opencode]]**
- `f96e2d4222` 2026-03-11 verbose catalog in system prompt, terse in the tool description.
- `c08fa5675f` 2026-04-05 "remove redundant Kimi skill section".
- `aacdb34e3f` 2026-06-07 "avoid duplicate skill catalog": tool description now static — "The skill name must match one of the skills listed in your system prompt" (`packages/opencode/src/tool/skill.txt:5`).

**Lesson** — Render a catalog in exactly one place; keep tool descriptions static.

Related: [[skill-progressive-disclosure]] · [[cache-stable-prompt-prefix]] · [[tool-description-design]] · [[opencode--skill-progressive-disclosure|opencode]]
