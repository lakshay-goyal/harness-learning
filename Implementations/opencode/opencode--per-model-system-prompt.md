---
type: implementation
harness: opencode
concept: per-model-system-prompt
commit: ecc4916b5a
files: [packages/opencode/src/session/system.ts:6-16, packages/opencode/src/session/system.ts:28-51, packages/opencode/src/session/llm/request.ts:56-66, packages/opencode/src/session/prompt/anthropic.txt, packages/opencode/src/session/prompt/gpt.txt, packages/opencode/src/session/prompt/codex.txt, packages/opencode/src/session/prompt/gpt-astra.txt, packages/opencode/src/session/prompt/beast.txt, packages/opencode/src/session/prompt/gemini.txt, packages/opencode/src/session/prompt/kimi.txt, packages/opencode/src/session/prompt/meta.txt, packages/opencode/src/session/prompt/trinity.txt, packages/opencode/src/session/prompt/default.txt, origin/v2:packages/core/src/plugin/optimize.ts:16-93]
---
[[per-model-system-prompt]] in [[opencode]].

## Mechanism
### Legacy runtime: selection
- `SystemPrompt.provider(model)` — ordered `includes()` on `model.api.id`, first match wins (`packages/opencode/src/session/system.ts:28-51`):

| test | prompt |
|---|---|
| `muse` | meta.txt with `{{MODEL_NAME}}` = Muse Glimmer / Muse Spark |
| `gpt-4` / `o1` / `o3` | beast.txt |
| `gpt` → `gpt-6` / `codex` / else | gpt-astra.txt / codex.txt / gpt.txt |
| `gemini-` | gemini.txt |
| `claude` | anthropic.txt |
| `trinity` (case-insensitive) | trinity.txt |
| `kimi` or provider `kimi-for-coding`/`moonshotai`/`moonshotai-cn` | kimi.txt |
| else | default.txt |
- An agent `prompt` replaces the family prompt entirely: `input.agent.prompt ? [agent.prompt] : SystemPrompt.provider(model)` (`packages/opencode/src/session/llm/request.ts:60`) → [[agent-profiles]], [[system-prompt-override]].
- Bare `o1`/`o3` substrings could misroute unrelated ids containing them (no incident found; unverified).
### Legacy runtime: what each prompt says (distinctive lines)
- **anthropic.txt** — Claude Code 2.0-style: "You are OpenCode, the best coding agent on the planet." (l.1); TodoWrite "VERY frequently" + "IMPORTANT: Always use the TodoWrite tool" (l.96); "CRITICAL that you use the Task tool instead of running search commands directly"; `<system-reminder>` "bear no direct relation…" (l.75).
- **gpt.txt** (non-codex GPT, modelled on Codex CLI) — "Use `multi_tool_use.parallel`… Never chain together bash commands with separators like `echo "====";`" (l.6); "Always use apply_patch for manual code edits" (l.27); "You struggle using the git interactive console"; no openers like "Done —".
- **codex.txt** — ASCII default; "Try to use apply_patch for single file edits" (l.8) yet "Use Read to view files, Edit to modify files" (l.12); question policy "Never ask permission questions like 'Should I proceed?'" (l.43-49).
- **gpt-astra.txt** (gpt-6, ported from origin/v2) — 46 lines; "Do not spawn subagents unless the user or applicable AGENTS.md/skill instructions explicitly ask" (l.46), the opposite of anthropic.txt's delegation push.
- **beast.txt** (gpt-4.x, o1, o3; community "Beast Mode") — "THE PROBLEM CAN NOT BE SOLVED WITHOUT EXTENSIVE INTERNET RESEARCH"; "Always read 2000 lines of code at a time"; markdown todo lists despite a todo tool; `.github/instructions/memory.instruction.md` memory (l.114).
- **gemini.txt** (modified gemini-cli) — "Most tool calls … will first require confirmation" (l.58), "/bug command" (l.62): never updated since import.
- **kimi.txt** — treat ambiguous requests as tasks; `<system-reminder>` = "authoritative system directives that you MUST follow" (l.17); update AGENTS.md when touched files are described there.
- **meta.txt** (Muse; written by a Meta engineer) — "Evidence before synthesis"; "every omitted line is a deletion" check before multi-line `edit` (l.33); computed results "should come from executed code, not copied text plus mental math" (l.54).
- **trinity.txt** (Arcee) — "Use exactly one tool per assistant message" (l.84); "Avoid repeating the same tool with the same parameters" (l.86).
- **default.txt** (fallback, formerly qwen.txt) — early Claude Code: "fewer than 4 lines", "DO NOT ADD ***ANY*** COMMENTS", no todo instructions.
- Orphans in tree: `copilot-gpt-5.txt` (routed only 2025-08-11 → 2025-09-01), `plan-reminder-anthropic.txt` (never imported; contains `/Users/aidencline/.claude/plans/happy-waddling-feigenbaum.md`, l.10) → [[borrowed-prompt-foreign-references]].
### origin/v2 branch
- One 14-line base `system.txt` + `OptimizePlugin`s hooked on `context`/`compaction`/`generate`: GPT (override; `astra` ids → gpt-astra), Anthropic (**append** one "Code comments" paragraph), Kimi, Arcee/Trinity, Meta; skipped when the agent has its own `system` (`origin/v2:packages/core/src/plugin/optimize.ts:16-93`) → [[minimal-system-prompt]].
- 2026-09-02 `8068c5e48c` codex.txt + gpt.txt merged; 2026-09-05 `4306c07b34` legacy Anthropic prompt removed; 2026-09-11 `9dd7149e75` Claude-only comment-density append → [[over-commenting-code]].

## Constants
| name | value | path:line |
|---|---|---|
| selectable prompts | 10 imports | `packages/opencode/src/session/system.ts:6-16` |

## Evolution (each switch = a model that misbehaved on the previous prompt)
- 2025-07-09 `1f6efc6b94` gpt-4.1 Beast prompt; 2025-07-15 `fdfd4d69d3` gemini-cli prompt; 2025-07-29 `df03e182d2` non-Claude get Anthropic prompt minus todo instructions.
- 2025-08-10 `13d3fba86b` gpt-5 → codex; 2025-08-11 `6eaa231587` → copilot-gpt-5; 2025-09-01 `4c261ab1db` → codex again.
- 2025-08-11 `dac1506680` Claude Code sync ("fewer than 4 lines"); 2025-09-30 `5a507023a6` scaled to query complexity → [[imperative-guideline-over-compliance]]; 2025-10-25 `795b845782` Claude Code 2.0 sync.
- 2025-11-07 `954a796b8a` polaris-alpha (stealth model) prompt; removed 2025-12-26 `25c68c8061`.
- 2026-01-20 `5622c53e1f` codex over-asking → [[excessive-permission-questions]].
- 2026-02-03 `015cd404e4` Trinity (reverted same day), 2026-02-06 `95ad6758af` re-added → [[model-cannot-parallel-tool-call]].
- 2026-03-18 `8ee939c741` fallback prompt trimmed (malware-refusal lines removed; motive unstated).
- 2026-03-19 `1ac1a0287c` "anthropic legal requests": the never-imported 166-line verbatim Claude Code prompt `anthropic-20250930.txt` (added `5a507023a6`) deleted with the bundled Claude auth plugin → [[provider-identity-shim]].
- 2026-03-20 `bfdc38e421` all `gpt*` → codex; 2026-03-26 `da1d37274f` own gpt.txt for non-codex GPT; 2026-03-28 `72cb9dfa31` minimised, file-reference format fixed → [[client-unrenderable-output-format]].
- 2026-03-31 `2daf4b805a` Kimi; 2026-08-12 `91df883231` Kimi by provider id.
- 2026-07-09 `b4665a8bf8` Muse; 2026-07-20 `5a8ee27254` Meta rewrite; 2026-08-10 `b9f3b382fc` all Muse ids.
- 2026-09-08 `5cd8e68fdd` GPT-6 Astra ported from origin/v2.

## Quirks / drift
- Prompt routing (`model.api.id`) and tool routing (`input.modelID`, [[model-specific-toolset]]) use different predicates → gpt-oss told to use apply_patch it lacks; codex told to use Edit it lacks → [[prompt-names-unavailable-tools]].
- `<system-reminder>` authority is framed four different ways across prompts → [[ephemeral-reminder-injection]].
- Prompts lifted from gemini-cli, VS Code Copilot, Claude Code and Codex keep foreign commands/paths → [[borrowed-prompt-foreign-references]].
- anthropic.txt has an empty `- ` bullet left by `795b845782` (l.72).

Contrast: pi ships one minimal prompt for all models and removed its short-lived Codex-specific variant → [[pi--minimal-system-prompt|pi]].
