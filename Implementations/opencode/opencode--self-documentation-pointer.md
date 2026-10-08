---
type: implementation
harness: opencode
concept: self-documentation-pointer
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt/anthropic.txt:12, packages/opencode/src/session/prompt/meta.txt:65, packages/opencode/src/skill/index.ts:26-33, packages/core/src/plugin/skill.ts:9-24]
---
[[self-documentation-pointer]] in [[opencode]].

## Mechanism
### Legacy runtime
- Pointer by URL, not path: anthropic.txt "When the user directly asks about OpenCode (eg. 'can OpenCode do…') … use the WebFetch tool to gather information … from OpenCode docs" at https://opencode.ai/docs (`packages/opencode/src/session/prompt/anthropic.txt:12`; also `meta.txt:65`). Other family prompts carry no pointer.
- Built-in skill `customize-opencode` (scoped "Use ONLY when the user is editing or creating opencode's own configuration…") loads real config schemas on demand, because "The model's intuition for what an opencode.json should look like is often wrong, and opencode hard-fails on invalid config" (`packages/opencode/src/skill/index.ts:26-33`) → [[skill-progressive-disclosure]].
### v2 runtime
- Same skill shipped as an embedded plugin skill at `/builtin/customize-opencode.md` (`packages/core/src/plugin/skill.ts:9-24`).
- origin/v2: per-prompt docs instruction deleted as redundant with the skill (`4d74854e8c` 2026-09-07 "trim redundant opencode instruction").

## Constants
| name | value | path:line |
|---|---|---|
| docs URL | https://opencode.ai/docs | `packages/opencode/src/session/prompt/anthropic.txt:12` |

## Evolution
- 2026-09-07 `4d74854e8c` (origin/v2) URL pointer removed in favour of the built-in skill.

## Quirks / drift
- The URL pointer depends on `webfetch`, which is always registered but may be denied by permissions.

Contrast: pi puts absolute local doc paths plus a topic map in the prompt, scoped to "only when the user asks about pi" → [[pi--self-documentation-pointer|pi]].
