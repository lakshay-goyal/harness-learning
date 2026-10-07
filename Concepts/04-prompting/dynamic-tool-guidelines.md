---
type: concept
stage: messages
tier: must-have
aliases: [promptSnippet, promptGuidelines, buildRules, "*ToolSystemPromptContribution", "<tools>", "<rules>", "${OPENCODE_TOOL_GUIDANCE}", SessionSystemPrompt.render, without_update_plan_instructions, update_plan_instructions.rs, "tools.update_plan.enabled", include_skills_usage_instructions, include_plugin_usage_instructions, include_apps_usage_instructions, BASE_INSTRUCTIONS_WITH_APPLY_PATCH]
harnesses: [pi, opencode, codex]
---
Each declared tool contributes its own one-line prompt snippet and usage guidelines; the rules block is generated from the tools actually declared on the request, deduplicated.

## Why
- A static rules list names tools that aren't there or omits ones that are → model prefers unavailable tools, claims read-only modes, or ignores new tools ([[prompt-names-unavailable-tools]]).
- Guidance about a tool belongs with the tool so plugins/extensions get first-class prompt presence without editing a central prompt.
- Without dedupe, two tools contributing the same rule (bash + PowerShell env hint) double it.

## Design space
- Static hand-written tool list + guidelines (pi v0, Oct 2025).
- Central conditional rules keyed on tool-set heuristics ("no bash/edit/write ⇒ READ-ONLY mode") — pi Nov 2025–Jan 2026; broken by extension tool overrides.
- **Per-tool `promptSnippet` + `promptGuidelines`, opt-in listing, set-dedupe, recomputed per request from declared (non-hidden) tools** — **pi chose** (Mar 2026 →).
- Fallback to API description when no snippet (pi tried; removed as duplicate bloat `7817e9b22`).
- No tool list in prompt at all; rely on API declarations (partially: pi lists only tools with snippets).
- **Inverse: fixed model-owned prompt; when a tool is disabled, strip the sections/bullets that mention it by literal heading/prefix match** (`## Planning`, `## \`update_plan\``, `- Use the plan tool `), only from harness-owned text, custom instructions untouched ✔ codex (`codex-rs/prompts/src/update_plan_instructions.rs`).
- Splice tool guidance into the base prompt at a string anchor (`base.replace("## Editing constraints", …)`) — codex 2025-11, silently dropped when the anchor was missing ([[anchor-based-prompt-injection-silently-fails]]); then append-to-family-prompt; then folded into per-model prompts.
- Per-model flags gating whole instruction fragments (`include_skills_usage_instructions`, `include_plugin_usage_instructions`, `include_apps_usage_instructions`) ✔ codex.
- Prompt variant per tool availability: base + apply_patch grammar for models without a native apply_patch tool (`BASE_INSTRUCTIONS_WITH_APPLY_PATCH`, codex until `a1abd53b6a`).
- Where behavioural guidance lives: in the system prompt for always-present tools (codex moved it *out* of `update_plan`'s description, `30ee24521b`) vs in the description of optional tools (spawn_agent, goal, plugin-install) ✔ codex.
- Central renderer keyed on present tool names, run after per-model tool curation (opencode origin/v2) vs per-tool contributions (pi).

## Implementations
- [[pi--dynamic-tool-guidelines|pi]] — `buildRules` order: shell rule → per-tool → extension → universal; hidden declarations excluded; skills hint reader-derived.
- [[codex--dynamic-tool-guidelines|codex]] — literal-heading stripping of update_plan guidance when the tool is off (default off since 2026-08-31); catalog flags gate skills/plugins/apps fragments; behavioural text moved out of always-present tool descriptions.
- [[opencode--dynamic-tool-guidelines|opencode]] — origin/v2 only: shell / write / edit guidance lines rendered into `${OPENCODE_TOOL_GUIDANCE}` when those tools are present, after per-model tool curation; legacy hard-codes tool advice in each family prompt.

## Failures
- [[prompt-names-unavailable-tools]]
- [[shell-cat-instead-of-read-tool]]
- [[skills-hidden-when-read-tool-absent]]
- [[imperative-guideline-over-compliance]]
- [[anchor-based-prompt-injection-silently-fails]]
- [[edits-bypass-patch-tool]]
- [[mode-state-confusion]]

## Related
[[minimal-system-prompt]] · [[tool-description-design]] · [[plugin-tools]] · [[deferred-tool-loading]] · [[guideline-softening]] · [[transcript-carried-system-prompt]] · [[per-model-system-prompt]] · [[task-list-tool]] · [[single-vs-per-model-system-prompt]]

## Tradeoffs
- [[single-vs-per-model-system-prompt]]
