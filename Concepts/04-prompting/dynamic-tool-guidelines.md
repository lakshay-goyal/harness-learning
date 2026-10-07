---
type: concept
stage: messages
tier: candidate
aliases: [promptSnippet, promptGuidelines, buildRules, "*ToolSystemPromptContribution", "<tools>", "<rules>"]
harnesses: [pi]
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

## Implementations
- [[pi--dynamic-tool-guidelines|pi]] — `buildRules` order: shell rule → per-tool → extension → universal; hidden declarations excluded; skills hint reader-derived.

## Failures
- [[prompt-names-unavailable-tools]]
- [[shell-cat-instead-of-read-tool]]
- [[skills-hidden-when-read-tool-absent]]
- [[imperative-guideline-over-compliance]]

## Related
[[minimal-system-prompt]] · [[tool-description-design]] · [[plugin-tools]] · [[deferred-tool-loading]] · [[guideline-softening]] · [[transcript-carried-system-prompt]]
