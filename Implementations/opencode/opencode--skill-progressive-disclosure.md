---
type: implementation
harness: opencode
concept: skill-progressive-disclosure
commit: ecc4916b5a
files: [packages/opencode/src/skill/index.ts:21-34, packages/opencode/src/skill/index.ts:175-230, packages/opencode/src/skill/index.ts:321-346, packages/opencode/src/skill/discovery.ts:67-126, packages/opencode/src/tool/skill.ts:27-60, packages/opencode/src/tool/skill.txt:5, packages/opencode/src/session/system.ts:107-119, packages/opencode/src/command/index.ts:140, packages/core/src/plugin/skill.ts:9-24, packages/core/src/skill/guidance.ts:46-68]
---
[[skill-progressive-disclosure]] in [[opencode]].

## Mechanism
### Legacy runtime
- Discovery order: `~/.claude/skills/**/SKILL.md` (unless `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS`) and `~/.agents/skills/**`; project `.claude`/`.agents` dirs walking up to the worktree; every config dir `{skill,skills}/**/SKILL.md`; `skills.paths` (`~/` expanded); `skills.urls` remote indexes cached with a `.opencode-version` file (`packages/opencode/src/skill/index.ts:21-25,175-230`; `packages/opencode/src/skill/discovery.ts:67-126`). `OPENCODE_DISABLE_EXTERNAL_SKILLS` turns off the `.claude`/`.agents` scan.
- Built-in skill `customize-opencode` ("Use ONLY when the user is editing or creating opencode's own configuration…") gives real config schemas instead of guesses (`skill/index.ts:26-33`) → [[self-documentation-pointer]].
- System prompt (verbose): "Skills provide specialized instructions… Use the skill tool to load a skill when a task matches its description." + `<available_skills><skill><name><description><location>` sorted by name; omitted entirely when `skill` is denied for the agent (`packages/opencode/src/session/system.ts:107-119`; `skill/index.ts:321-338`). Code comment: agents "ingest the information about skills a bit better if we present a more verbose version of them here and a less verbose version in tool description".
- Loading is a **dedicated tool**, not a file read: `skill(name)` asks permission `skill` on the name, returns `<skill_content name>` with the SKILL.md body, base dir and `<skill_files>` = up to 10 sampled sibling files via ripgrep (`packages/opencode/src/tool/skill.ts:27-60`).
- Skill tool description is static: "The skill name must match one of the skills listed in your system prompt" (`packages/opencode/src/tool/skill.txt:5`).
- Skills are also slash commands (`source: "skill"`, `packages/opencode/src/command/index.ts:140`) → [[prompt-template-expansion]].
- No name/description length validation found (absence).
### v2 runtime
- Per-turn skill guidance lists only permission-permitted names + descriptions, **no paths** (`packages/core/src/skill/guidance.ts:46-68`); `customize-opencode` shipped as an embedded plugin skill at `/builtin/customize-opencode.md` (`packages/core/src/plugin/skill.ts:9-24`).
- Missing-skill errors stopped enumerating the unfiltered catalog (`3f64b5e621`, `specs/v2/schema-changelog.md:813`) (candidate 07-safety failure `error-message-leaks-filtered-catalog`).

## Constants
| name | value | path:line |
|---|---|---|
| sampled bundled files | 10 | `packages/opencode/src/tool/skill.ts:42`; `packages/core/src/tool/skill.ts:15` |

## Evolution
- 2025-12-29 `ef8388f0ee` revert of reading global `~/.claude/skills` (later re-added).
- 2026-03-11 `0f6bc8ae71` presentation changed "to increase likelyhood of skill invocations"; same day `f96e2d4222` "a little less token heavy".
- 2026-04-05 `c08fa5675f` redundant skill section removed from kimi.txt; 2026-06-07 `aacdb34e3f` "avoid duplicate skill catalog" (tool-description copy dropped) → [[duplicated-catalog-in-prompt]].
- 2026-06-05 `3f64b5e621` v2 skill guidance admitted.

## Quirks / drift
- gpt-astra.txt: "Do not use a skill based solely on keywords, superficial relevance" (l.7) — a brake on over-invocation that the Claude prompts lack.

Contrast: pi lists name/description/path and the model reads the file with its read tool → [[pi--skill-progressive-disclosure|pi]].
