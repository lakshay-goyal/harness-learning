---
type: concept
stage: messages
tier: variant
aliases: [Personality, model_personality, "personality = none", friendly, pragmatic, "{{ personality }}", "<personality_spec>", PERSONALITY_SECTION_HEADER, strip_personality_section, supports_personality]
harnesses: [codex]
---
A swappable persona/tone section of the system prompt, selected by a user setting and kept separate from behavioural rules, so voice can change without touching the rules.

## Why
- Users disagree about tone (cheerful vs terse); hard-coding one voice into the rules forces forks or custom prompts.
- Tone is the part of a prompt most likely to be copied into output; isolating it makes it removable.
- Counter-evidence: codex shipped Friendly/Pragmatic selection and retired it after 8 months — voice ended up a model-owned property ([[no-user-selectable-personality]]).

## Design space
- No persona section; voice implicit in rules ✔ pi.
- **Template placeholder `{{ personality }}` filled from per-persona files** ✔ codex 2026-01-20 → 2026-09-12.
- Mid-session switch as an appended developer message (`<personality_spec>`) instead of rewriting the cached system prompt ✔ codex (`8b3521ee77`, retired `132c739171`).
- **Opt-out only: strip the `# Personality` section from the model's catalog text** (`personality = "none"`) ✔ codex today.
- One fixed voice per model, shipped in the catalog prompt ("You have a vivid inner life as Codex…", gpt-5.5) ✔ codex today → [[per-model-system-prompt]].

## Implementations
- [[codex--personality-variants|codex]] — `{{ personality }}` + friendly/pragmatic files (2026-01) → `<personality_spec>` mid-session (2026-01) → retired (2026-09); only `personality = "none"` H1-section stripping survives.

## Failures
- (none recorded; retirement rationale only in commit subjects)

## Related
[[per-model-system-prompt]] · [[message-role-layering]] · [[harness-identity]] · [[system-prompt-override]] · [[transcript-carried-system-prompt]] · [[single-vs-per-model-system-prompt]]
