---
type: failure
concepts: [self-documentation-pointer, skill-progressive-disclosure]
harnesses: [codex]
---
**Symptom** — Questions phrased as "you / your / this app" about Codex did not trigger the bundled docs skill; the model answered from stale parametric knowledge.

**Root cause** — Self-docs are reached only through skill-description matching; the description named the product ("Codex", "OpenAI products or APIs") but not the deictic forms users actually type.

**Fix · [[codex]]** — `a4ed6c5aa0` 2026-05-28 added "asks about Codex itself or choosing Codex surfaces… use the Codex manual helper first for broad Codex self-knowledge"; `a5082373f1` 2026-07-29 rewrote the openai-docs description: "Use for Codex models/pricing, scheduled tasks, skills, settings, setup, troubleshooting, customization, automations, and self-knowledge—including 'you,' 'your,' 'this app,' or 'this coding agent' when they refer to Codex" (`codex-rs/skills/src/assets/samples/openai-docs/SKILL.md` frontmatter).

**Lesson** — Skill descriptions are matched against user phrasing; self-referential questions need the deictic forms listed explicitly (or an unconditional scope rule, as pi's "when the user asks about pi").

Related: [[self-documentation-pointer]] · [[skill-progressive-disclosure]] · [[codex--self-documentation-pointer|codex]]
