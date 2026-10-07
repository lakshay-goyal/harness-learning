---
type: absence
harnesses: [codex]
---
# no-user-selectable-personality

Friendly/Pragmatic personality selection was retired; voice is a model-owned property, not a user knob.

**What's missing**
- Only `personality = "none"` remains (strips the `# Personality` H1 section).

**Evidence of decision**
- Shipped 2026-01-20 `714151eb4e` (`{{ personality }}` template placeholder); `c18277043e` 2026-09-11 fixed friendly text embedded into gpt-5.4/5.5; retired 2026-09-12 `132c739171` "Retire Friendly and Pragmatic personality selection (#44946)" ("Stop emitting `<personality_spec>` developer messages"); `d63a9b8344` 2026-10-06 removed `instructions_variables` metadata. Feature key `personality` is `Stage::Removed` ([[feature-flag-stages]]).

**Implication**
- Prompt variants per user preference multiply the per-model prompt matrix; codex collapsed it ([[personality-variants]], [[per-model-system-prompt]]).

Related: [[personality-variants]] · [[per-model-system-prompt]] · [[single-vs-per-model-system-prompt]] · [[Absences]]
