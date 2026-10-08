---
type: implementation
harness: codex
concept: personality-variants
commit: 622e9e3696
files: [codex-rs/models-manager/src/model_info.rs:17, codex-rs/models-manager/src/model_info.rs:61]
---
[[personality-variants]] in [[codex]].

## Mechanism
- Today: only the opt-out remains. `personality = "none"` strips the `# Personality` section up to the next H1 from the model's catalog `instructions_template` (`PERSONALITY_SECTION_HEADER = "# Personality"`, `codex-rs/models-manager/src/model_info.rs:17`, `:61-95`). No `<personality_spec>` developer messages are emitted; `instructions_variables` metadata removed (`d63a9b8344`).
- Voice now lives inside each model's catalog prompt (e.g. gpt-5.5 "You have a vivid inner life as Codex: intelligent, playful, curious, and deeply present.", added `c10f95ddac` 2026-04-24) → [[codex--per-model-system-prompt]].

## Evolution
- 2026-01-20 `714151eb4e` "feat(personality) introduce model_personality config (#9459)": gpt-5.2-codex instructions template with `{{ personality }}` (`a1abd53b6a^:codex-rs/core/templates/model_instructions/gpt-5.2-codex_instructions_template.md:3`) filled from `templates/personalities/gpt-5.2-codex_friendly.md` / `_pragmatic.md`. Pragmatic text: "You are a deeply pragmatic, effective software engineer... avoiding cheerleading, motivational language, or artificial reassurance" (`a1abd53b6a^:codex-rs/core/templates/personalities/gpt-5.2-codex_pragmatic.md`).
- 2026-01-22 `8b3521ee77` "feat(core) update Personality on turn (#9644)": mid-session switch sent as a `<personality_spec>` developer message so the cached system prompt is untouched.
- 2026-07-10 `09ccae2c07` "Honor `personality = "none"`": H1-section stripping from catalog text.
- 2026-09-11 `c18277043e` fixed friendly text embedded into gpt-5.4/5.5.
- 2026-09-12 `132c739171` "Retire Friendly and Pragmatic personality selection (#44946)": "Stop emitting `<personality_spec>` developer messages".
- 2026-10-06 `d63a9b8344` removes `instructions_variables` metadata.

## Quirks
- Life cycle ~8 months; no rationale beyond commit subjects (unverified why). Absence note: [[no-user-selectable-personality]].
- `instructions_template` keeps its name "for model catalog compatibility" although it is now literal text (`codex-rs/protocol/src/openai_models.rs:540-543`).

## Versus pi
- pi has no persona knob; tone is whatever the minimal prompt + model default produce. Codex converged to the same end state for users (no selectable voice) but via model-owned voice text rather than an absent section.
