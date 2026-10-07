---
type: concept
stage: messages
tier: candidate
aliases: [system-prompt.ts, buildSystemPrompt, buildSystemPromptSections, "default system prompt"]
harnesses: [pi]
---
Keep the harness-authored system prompt tiny; push behavior into tool contracts, on-demand docs, and environment rather than standing instructions.

## Why
- Every standing rule is paid on every request and is a cache-prefix liability ([[cache-stable-prompt-prefix]]).
- Rules written against one model's habit go stale; pi's history is a graveyard of them (identity override, "do NOT use cat to display", READ-ONLY mode, "Prefer grep/find/ls") — see [[forced-model-identity-override]], [[model-echoes-work-via-shell]], [[prompt-names-unavailable-tools]].
- Rules that mention absent tools or volatile facts actively mislead ([[prompt-names-unavailable-tools]], [[volatile-system-prompt-prefix]]).
- Without a minimal core the prompt grows by accretion anyway: pi grew ~4× (≈175 → ≈680 tok), mostly self-docs.

## Design space
- **Tiny harness core + contributed sections** — preamble + 2 universal rules; tools, docs, context, skills contribute their own text. **pi chose** (`system-prompt.ts:128-193`).
- Large monolithic prompt with workflows, examples, tone rules (typical of other harnesses; pi rejected).
- Provider-specific prompt variants (pi tried a Codex bridge + static allowlisted instructions in Jan 2026, removed within 2 weeks → [[harness-identity]]).
- Offload long reference to docs read on demand ([[self-documentation-pointer]]; codemode reference moved to `docs/codemode.md`, 5.3k → 3.3k tok request).
- Volatile facts via env/tools instead of prompt text ([[env-vars-as-context]]).

## Implementations
- [[pi--minimal-system-prompt|pi]] — ~680 tok default; preamble + `<tools>/<rules>/<docs>/<project_context>/<skills>/<cwd>` sections; full 45-step timeline + 19 removed rules.

## Failures
- [[model-echoes-work-via-shell]]
- [[forced-model-identity-override]]
- [[windows-backslash-cwd-copied-into-shell]]
- [[partial-file-read-acted-on]]
- [[prompt-names-unavailable-tools]]
- [[volatile-system-prompt-prefix]]

## Related
[[dynamic-tool-guidelines]] · [[transcript-carried-system-prompt]] · [[xml-prompt-boundaries]] · [[guideline-softening]] · [[tool-description-design]] · [[minimal-default-toolset]] · [[no-date-in-prompt]]
