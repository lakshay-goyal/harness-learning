---
type: failure
concepts: [dynamic-tool-guidelines, minimal-system-prompt]
harnesses: [pi]
---
**Symptom** — System-prompt rules referenced or implied tools the request didn't declare (or mis-described the toolset):
- "Prefer grep/find/ls tools over bash for file exploration" emitted when *any* of grep/find/ls was present → model preferred unavailable ones (#5132).
- "You are in READ-ONLY mode - you cannot modify files or execute arbitrary commands" inferred from absence of bash/edit/write — wrong once extensions could add/override tools.
- "Use bash ONLY for read-only operations…" — prompt-only restriction with no enforcement.
- Extension tools listed via "Custom tool" fallback / description duplicating API declarations.
- `codemode.mode:"only"` listed read/bash/edit/write that requests didn't declare (#10192); `prepareLoadout`-hidden tools named in rules and the skills hint (#10343); example extensions stripped built-in guidelines (#10072 CHANGELOG `:209`; #10193 `ee602414c`).

**Root cause** — Rules derived from heuristics over the *selected* tool set rather than the exact declarations the request carries; tool guidance centralized instead of owned per tool.

**Fix · [[pi]]**
- `e3dd4f21d` 2026-01-08: READ-ONLY rule removed (with extension tool overrides).
- `b846a4bfc` 2026-01-20 (#645): bash read-only rule removed silently.
- `73734a23a` 2026-01-23: only built-ins listed ("Extension tools are already described via API tool definitions"); `7817e9b22` 2026-03-17 (#2285): snippets opt-in, no description fallback.
- `1ab289980` 2026-05-28 (#5132): "Prefer grep/find/ls…" removed (CHANGELOG `:1806` "avoid preferring unavailable file exploration tools"; `system-prompt.ts:115` at the time).
- `028c0ec56` 2026-09-30 (#10192): codemode-hidden tools not listed (diff filtered `toolSnippets` by `_hiddenDeclarations` in `agent-session.ts`; superseded at HEAD by `hiddenTools`, `agent-session.ts:1700-1706, 1742-1743`).
- `c30840c2e` 2026-10-05 (#10343): `hiddenTools` excluded from tool list and rules; skills hint "indirect" (HEAD `system-prompt.ts:150, 174-178`; `agent-session.ts:1742-1743` "The tool list and rules must match the declarations the request carries").

**Lesson** — Generate every tool-related sentence from the exact declarations sent on that request, and let each tool own its guidance; never infer permissions from tool absence.

Related: [[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[deferred-tool-loading]] · [[code-mode]] · [[skills-hidden-when-read-tool-absent]] · [[tool-loadout-stale-within-run]] · [[pi--dynamic-tool-guidelines|pi]]
