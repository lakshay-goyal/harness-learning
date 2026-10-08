---
type: concept
stage: messages
tier: must-have
aliases: [SystemPrompt.provider, PROMPT_ANTHROPIC, PROMPT_BEAST, PROMPT_CODEX, PROMPT_GPT, PROMPT_ASTRA, PROMPT_GEMINI, PROMPT_KIMI, PROMPT_TRINITY, PROMPT_META, OptimizePlugin, model-specific-system-prompt, base_instructions, BASE_INSTRUCTIONS_DEFAULT, model_messages.instructions_template, ModelMessages, ResolvedModelMessages, find_model_info_for_slug, "per-model-family prompts", model-owned-system-prompt]
harnesses: [opencode, codex]
---
The harness ships several base system prompts and picks one per request from the model id or provider.
The base system prompt is owned by the model entry in a (remotely updatable) model catalog, not by harness code: each model family ships its own tuned text, with a generic fallback for unknown models.

- Model families respond differently to the same rules: brevity, todo usage, delegation, parallel calls and question-asking all need per-family tuning (see Failures).
- Vendors and model labs contribute prompts tuned to their own model (Meta, Arcee, Moonshot in opencode).
- Cost: N prompts drift independently, copy foreign references from the harnesses they were lifted from ([[borrowed-prompt-foreign-references]]), and must stay consistent with the per-model toolset ([[model-specific-toolset]]).
## Why
- A model trained inside the harness needs far less standing instruction than a general model: codex-tuned prompts are ~6.6–7.6 KB vs 20.9 KB for the generic base (`916fdc2a37`); one prompt for all models over- or under-instructs most of them.
- Each new model generation re-broke behaviour the old prompt had fixed (bias-to-action, permission asking, verbosity) → per-model text is the patch surface ([[premature-turn-end]], [[over-validation-in-interactive-mode]]).
- Shipping the prompt with model metadata lets the vendor change a model's prompt without a client release.
- Costs: prompt text drifts away from harness code (stale limits, [[prompt-states-stale-harness-limits]]); large rewrites silently drop load-bearing lines ([[prompt-rewrite-drops-load-bearing-lines]]); prompt files become code that formatters corrupt ([[formatter-corrupts-prompt-file]]).

- One prompt for all models (pi, [[minimal-system-prompt]]).
- **Ordered substring chain on the model id, first match wins, full prompt per family** (opencode legacy, 10 prompts).
- **Small shared base + per-family plugin that overrides or appends** (origin/v2 `OptimizePlugin`: GPT/Kimi/Arcee/Meta override, Anthropic appends one paragraph).
- Provider-id fallback when the model id is opaque (opencode Kimi via `moonshotai*` providers).
- Agent prompt replaces the family prompt entirely (opencode legacy and origin/v2: skipped when the agent has its own system).
- Templated model name inside a shared family prompt (opencode `{{MODEL_NAME}}` for Muse).
## Design space
- One harness-authored prompt for every model ✔ pi ([[minimal-system-prompt]]); codex 2025-04 → 2025-09 (`31d0d7a305`).
- Hard-coded per-family selection in client code (`starts_with` slug chain → `include_str!` prompt files) — codex 2025-09-14 `916fdc2a37` → removed 2026-02-09 `a1abd53b6a`.
- **Catalog-owned prompt**: `model_messages.instructions_template` per model in bundled + remote `models.json`; unknown slug → generic fallback prompt ✔ codex (2026-02 →).
- Per-model *fragments* beyond the base (approvals, permissions, collaboration modes, multi-agent, token budget, persistent mode, content-filter guidance) also catalog-owned ✔ codex; missing → bundled default, explicit empty → suppress (`a8c36ca6d2`).
- Template variables inside the catalog text (`{{ personality }}`) ✔ codex 2026-01 → retired 2026-10 ([[personality-variants]]).
- User replacement of the model's prompt: `base_instructions` / `model_instructions_file` ✔ codex ("STRONGLY DISCOURAGED") → [[system-prompt-override]].
- Prompt names the model ("You are GPT-5.1 running in the Codex CLI") — possible only when the prompt is per-model ✔ codex vs never override model identity ✔ pi → [[harness-identity]].

- [[opencode--per-model-system-prompt|opencode]] — `SystemPrompt.provider(model)`: muse → meta, gpt-4/o1/o3 → beast, gpt-6 → astra, codex → codex, gpt → gpt, gemini- → gemini, claude → anthropic, trinity, kimi → kimi, else default.
## Implementations
- [[codex--per-model-system-prompt|codex]] — catalog `models.json` carries 17–22 KB `instructions_template` per model; fallback `codex-rs/models-manager/prompt.md` (20.9 KB) for unknown slugs; 2025-09 → 2026-02 per-family prompt files in client; ~20 single-line behavioural patches in history.

- [[edit-oldstring-drops-lines]]
- [[borrowed-prompt-foreign-references]]
- [[autonomy-prompt-overreach]]
- [[excessive-permission-questions]]
- [[client-unrenderable-output-format]]
- [[model-cannot-parallel-tool-call]]
- [[over-commenting-code]]
- [[todo-tool-usage-calibration]]
- [[imperative-guideline-over-compliance]]
- [[prompt-names-unavailable-tools]]
## Failures
- [[prompt-states-stale-harness-limits]] (05-context)
- [[formatting-instruction-echoed-literally]]
- [[client-unrenderable-output-format]]
- [[premature-turn-end]]
- [[over-validation-in-interactive-mode]]
- [[prompt-rewrite-drops-load-bearing-lines]]
- [[formatter-corrupts-prompt-file]]
- Cross-group: [[foreign-harness-tool-hallucination]] (03-tools)

[[minimal-system-prompt]] · [[model-specific-toolset]] · [[harness-identity]] · [[provider-identity-shim]] · [[system-prompt-override]] · [[agent-profiles]] · [[ephemeral-reminder-injection]]
## Related
[[minimal-system-prompt]] · [[model-catalog]] · [[personality-variants]] · [[harness-identity]] · [[system-prompt-override]] · [[message-role-layering]] · [[dynamic-tool-guidelines]] · [[guideline-softening]] · [[single-vs-per-model-system-prompt]]

## Tradeoffs
- [[single-vs-per-model-system-prompt]]
