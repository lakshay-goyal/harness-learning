---
type: implementation
harness: opencode
concept: minimal-system-prompt
commit: ecc4916b5a
files: [packages/opencode/src/session/llm/request.ts:56-66, packages/opencode/src/session/prompt.ts:1262-1271, packages/core/src/plugin/agent.ts:12-13, origin/v2:packages/core/src/session/runner/prompt/system.txt:1-15, origin/v2:packages/core/src/session/system-prompt.ts:5-28, origin/v2:packages/core/src/plugin/optimize.ts:16-93]
---
[[minimal-system-prompt]] in [[opencode]]. Legacy is the opposite pole; origin/v2 converges on it.

## Mechanism
### Legacy runtime
- Final system = **one string**: (agent prompt ?? 46–155-line family prompt) + env block + instruction files + `<mcp_instructions>` + skills catalog (+ structured-output line) + per-message `user.system` (`packages/opencode/src/session/llm/request.ts:56-66`; `packages/opencode/src/session/prompt.ts:1262-1271`) → [[per-model-system-prompt]].
- Family prompts carry workflows, examples, tone rules, git protocols and tool advice that duplicates tool descriptions.
### v2 runtime (dev)
- Build agent system is one sentence: "You are an AI coding agent. Help the user accomplish software engineering tasks by inspecting the workspace, making targeted changes, and using tools according to the configured permissions." (`packages/core/src/plugin/agent.ts:12-13`).
### origin/v2 branch
- 14-line `system.txt`: identity line; `# Harness` (GFM rendering, `<system-reminder>` authority, prefer parallel calls, `${OPENCODE_TOOL_GUIDANCE}`); `# Communication`; `# Working in codebases` incl. "Treat unfamiliar files or changes as potential user work and investigate before deleting or overwriting them." (`origin/v2:packages/core/src/session/runner/prompt/system.txt:1-15`).
- Tool guidance lines rendered only for tools present → [[dynamic-tool-guidelines]].
- Per-family prompts become plugins (override or append) → [[per-model-system-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| origin/v2 base prompt | 15 lines | `origin/v2:packages/core/src/session/runner/prompt/system.txt` |
| legacy anthropic.txt | 105 lines | `packages/opencode/src/session/prompt/anthropic.txt` |

## Evolution
- 2026-05-16 `548648a3d9` shell/todowrite/task descriptions cut to ~20–30% of size.
- 2026-08-14 `8640ea3374` "minimize system prompt" (origin/v2): 95-line base → 14-line system.txt.
- 2026-09-02 `a7b8174917` GPT autonomy section replaced with scope guidance ("Do not infer authorization for work beyond the user's request") → [[autonomy-prompt-overreach]].
- 2026-09-05 `4306c07b34` legacy Anthropic prompt removed (origin/v2); Claude gets base + one comment-density paragraph (`9dd7149e75`).
- 2026-09-07 `4d74854e8c` "trim redundant opencode instruction".

## Quirks / drift
- Dev ships both extremes: legacy family prompts up to 155 lines and a one-sentence v2 build prompt.

Contrast: pi keeps a tiny core with contributed sections and rejected provider-specific variants → [[pi--minimal-system-prompt|pi]].
