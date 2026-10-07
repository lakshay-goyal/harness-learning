---
type: concept
stage: messages
tier: must-have
aliases: [system-prompt.ts, buildSystemPrompt, buildSystemPromptSections, "default system prompt", base_instructions/default.md, models-manager/prompt.md]
harnesses: [pi, opencode, codex]
---
Keep the harness-authored system prompt tiny; push behavior into tool contracts, on-demand docs, and environment rather than standing instructions.

## Why
- Every standing rule is paid on every request and is a cache-prefix liability ([[cache-stable-prompt-prefix]]).
- Rules written against one model's habit go stale; pi's history is a graveyard of them (identity override, "do NOT use cat to display", READ-ONLY mode, "Prefer grep/find/ls") — see [[forced-model-identity-override]], [[model-echoes-work-via-shell]], [[prompt-names-unavailable-tools]].
- Rules that mention absent tools or volatile facts actively mislead ([[prompt-names-unavailable-tools]], [[volatile-system-prompt-prefix]]).
- Without a minimal core the prompt grows by accretion anyway: pi grew ~4× (≈175 → ≈680 tok), mostly self-docs.

## Design space
- **Tiny harness core + contributed sections** — preamble + 2 universal rules; tools, docs, context, skills contribute their own text. **pi chose** (`system-prompt.ts:128-193`).
- Large monolithic prompt with workflows, examples, tone rules (typical of other harnesses; pi rejected) — ✔ codex (contrast): 20.9 KB / 275-line fallback with planning examples, preamble rules, final-answer formatting spec; 17–22 KB per catalog model.
- Size by model fit: short prompt (~6.6–7.6 KB) for models trained on the harness, long for general models ✔ codex (`916fdc2a37`) → [[per-model-system-prompt]].
- Keep the big prompt *static* and move volatile policy out into appended developer fragments (permissions, modes, time) ✔ codex (`87f7226cca` deleted ~35 lines of sandbox text from every static prompt) → [[message-role-layering]], [[world-state-diff-injection]].
- Provider-specific prompt variants (pi tried a Codex bridge + static allowlisted instructions in Jan 2026, removed within 2 weeks → [[harness-identity]]).
- Offload long reference to docs read on demand ([[self-documentation-pointer]]; codemode reference moved to `docs/codemode.md`, 5.3k → 3.3k tok request).
- Volatile facts via env/tools instead of prompt text ([[env-vars-as-context]]).
- Small shared base + per-family override/append plugins (opencode origin/v2) → [[per-model-system-prompt]].

## Implementations
- [[pi--minimal-system-prompt|pi]] — ~680 tok default; preamble + `<tools>/<rules>/<docs>/<project_context>/<skills>/<cwd>` sections; full 45-step timeline + 19 removed rules.
- [[codex--minimal-system-prompt|codex]] — the opposite: ~5k-token standing prompt per model; size kept in check by stripping disabled-tool sections, moving policy to dynamic fragments and deleting stale limits.
- [[opencode--minimal-system-prompt|opencode]] — legacy: 46–155-line family prompts joined with env, instructions, MCP and skills into one string; v2 dev build agent is one sentence; origin/v2: 15-line base + tool guidance + per-family plugins.

## Failures
- [[model-echoes-work-via-shell]]
- [[forced-model-identity-override]]
- [[windows-backslash-cwd-copied-into-shell]]
- [[partial-file-read-acted-on]]
- [[prompt-names-unavailable-tools]]
- [[volatile-system-prompt-prefix]]
- [[anchor-based-prompt-injection-silently-fails]]
- [[mode-state-confusion]]
- [[prompt-states-stale-harness-limits]] (05-context)
- [[autonomy-prompt-overreach]]

## Related
[[dynamic-tool-guidelines]] · [[transcript-carried-system-prompt]] · [[xml-prompt-boundaries]] · [[guideline-softening]] · [[tool-description-design]] · [[minimal-default-toolset]] · [[no-date-in-prompt]] · [[per-model-system-prompt]] · [[single-vs-per-model-system-prompt]]

## Tradeoffs
- [[single-vs-per-model-system-prompt]]
