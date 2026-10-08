---
type: implementation
harness: opencode
concept: harness-package-distribution
commit: ecc4916b5a
files: [packages/opencode/src/plugin/shared.ts:172-216, packages/core/src/npm.ts:86-100, packages/opencode/src/skill/index.ts:186-230, packages/opencode/src/skill/discovery.ts:35-118]
---
[[harness-package-distribution]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Plugins as npm packages**: listed by spec in config `plugin` (`name@version`, `file://`, relative or absolute path) (`packages/opencode/src/plugin/shared.ts:172-180`); npm specs installed into the harness cache with arborist, `ignoreScripts: true`, `savePrefix: ""` under a lock (`packages/core/src/npm.ts:86-100`); package `engines.opencode` must match the running version (`packages/opencode/src/plugin/shared.ts:200`). No dedicated `install` CLI; editing config is the install.
- **Skills from many roots** (scan order): `~/.claude/skills/**/SKILL.md` and `~/.agents/skills/**`; project `.claude`/`.agents` dirs walking up to the worktree; every config dir `{skill,skills}/**/SKILL.md`; `skills.paths`; `skills.urls` (`packages/opencode/src/skill/index.ts:186-230`). Kill switches `OPENCODE_DISABLE_EXTERNAL_SKILLS`, `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS` (`packages/opencode/src/effect/runtime-flags.ts:21,29`).
- **Remote skill indexes** (`skills.urls`): fetch `<url>/index.json` (skills with `name`, `files`, `version`), download files into `<cache>/skills/<name>`; when `version` changes, download into a staging dir, require `SKILL.md`, swap atomically with backup; version recorded in `.opencode-version` (`packages/opencode/src/skill/discovery.ts:35-118`). No signature or hash check.
- **Built-in skill**: `customize-opencode` ships in core and is only for editing opencode's own config (`packages/opencode/src/skill/index.ts:34`).
- Cross-harness compatibility is the distribution strategy: Claude Code skill dirs and `.agents/skills` are read as-is.

### v2 runtime
- v2 config drops user-authored `command` in favor of skills and replaces `skills: {paths?, urls?}` with a single array (`specs/v2/config.md:43-50`).

## Constants
| name | value | path:line |
|---|---|---|
| remote skill version marker | `.opencode-version` | `packages/opencode/src/skill/discovery.ts:80` |

## Evolution
- 2025-12-29 `ef8388f0ee` reverted "read global ~/.claude/skills"; present again at HEAD (`packages/opencode/src/skill/index.ts:186-193`).

## Quirks / drift
- Reading `~/.claude/skills` means another harness's installed skills change opencode's behavior without opt-in.

pi contrast: `pi install npm:|git:|https:|./path` with a package manifest, filters and pinning ([[pi--harness-package-distribution|pi]]).
