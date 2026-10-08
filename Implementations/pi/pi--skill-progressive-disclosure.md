---
type: implementation
harness: pi
concept: skill-progressive-disclosure
commit: b30a6dd77
files: [packages/coding-agent/src/core/skills.ts:10-16, packages/coding-agent/src/core/skills.ts:160-275, packages/coding-agent/src/core/skills.ts:347-388, packages/coding-agent/src/core/skills.ts:424-455, packages/coding-agent/src/core/system-prompt.ts:174-182, packages/coding-agent/src/core/agent-session.ts:2146-2160, packages/coding-agent/src/core/package-manager.ts:468-487]
---
[[skill-progressive-disclosure]] in [[pi]].

## Mechanism
- **Discovery** (`packages/coding-agent/src/core/skills.ts:160-275`): a dir containing `SKILL.md` is a skill root (no further recursion, `:164,195`); root-level `.md` files only in the top dir; recurse subdirs; skip dot-dirs and `node_modules` (`:228-229`); honor `.gitignore/.ignore/.fdignore` (`:16`; `f89b49bae`). Non-`SKILL.md` markdown without description silently ignored (`:306-308`; `8c2529dae` #8012).
- **Locations**: `~/.pi/agent/skills`, `<cwd>/.pi/skills` (trusted only), `~/.agents/skills`, ancestor `.agents/skills` from cwd up to git root (trusted only) (`package-manager.ts:468-487, 2458-2560`; `docs/skills.md:61`) → [[project-trust-gate]]. Extensions can add paths via `resources_discover` ([[extension-event-hooks]]).
- **Validation** per Agent Skills spec: name ≤64 (`MAX_NAME_LENGTH`, `:11`), `^[a-z0-9-]+$`, no leading/trailing/double hyphen; description required, ≤1024 (`:14`, `:92-127`). Violations are **warnings, still loaded**; only missing description blocks (`:121, 329-332`). Name falls back to parent dir (`:311-321`; `5d31e70b8`).
- **Collision**: first discovered wins + `collision` diagnostic; symlink dupes deduped by realpath silently (`:424-455`).
- **Prompt listing** `formatSkillsForPrompt(skills, fileReadTool)` (`:347-388`): excludes `disableModelInvocation` skills; intro "The following skills provide specialized instructions for specific tasks."; reader-specific hint: "Use the read tool to load a skill's file when the task matches its description." / "Use bash to load a skill's file…" / "Load a skill's file…" (indirect); relative-path rule "When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands." (`:372`); `<available_skills><skill><name/><description/><location/></skill></available_skills>` with XML escaping (`escapeXml`). Body never in prompt.
- **Reader gating** (`system-prompt.ts:174-182`): reader = first of `read`, `bash` in declared tools; else `"indirect"` if a reader is selected but hidden; else skills section omitted entirely.
- **Explicit invocation** `/skill:name args` → user message `<skill name="…" location="…">\nReferences are relative to <baseDir>.\n\n<body>\n</skill>` + `\n\n<args>` (`agent-session.ts:2146-2160`); works even for `disable-model-invocation: true` skills (`951fb953e` #927). `enableSkillCommands` (default true) controls command discovery only (`settings-manager.ts:164, 1288`; `docs/skills.md:53`).
- Section `skills` is patchable ([[transcript-carried-system-prompt]]); `/reload` reloads skills (`slash-commands.ts:42`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_NAME_LENGTH` | 64 | `skills.ts:11` (`05b7b8133`) |
| `MAX_DESCRIPTION_LENGTH` | 1024 | `skills.ts:14` (`05b7b8133`) |
| `IGNORE_FILE_NAMES` | .gitignore, .ignore, .fdignore | `skills.ts:16` |
| per-skill prompt cost | ~4 lines + description | `skills.ts:377-383` |

## Evolution
- `09bca9672` 2025-12-12 (#171) "skills system with Claude Code compatibility": "Use the read tool to load a skill's file when the task matches its description. Skills may contain {baseDir} placeholders…".
- `05b7b8133` 2025-12-19 "Skills standard compliance": XML `<skill><name><description><location>`; `{baseDir}` sentence dropped; spec limits.
- `951fb953e` 2026-01-24 (#927): `disable-model-invocation` frontmatter.
- `3c687b427` 2026-02-01 (#1136): path-resolution guidance in preamble; `5d6a7d6c3` 2026-02-02 (#1171) better skill dir resolution.
- `f89b49bae` 2026-02-05: respect ignore files.
- `a8e3c829f` 2026-03-14 (#2075): stop recursion at skill root.
- `5d31e70b8` 2026-05-16: names may differ from directories.
- `8c2529dae` 2026-08-17 (#8012): don't load root mds as skills from settings paths.
- `1d6dbf9e3` 2026-09-03 (#8552): bash-only toolsets keep skills ("Use bash to load…").
- `9e05370b2` 2026-09-16: `<skills>` section tag.
- `c30840c2e` 2026-10-05 (#10343): hidden reader → "indirect" hint naming no tool.

## Evidence commits
`09bca9672`, `05b7b8133`, `951fb953e`, `3c687b427`, `5d6a7d6c3`, `f89b49bae`, `a8e3c829f`, `5d31e70b8`, `8c2529dae`, `1d6dbf9e3`, `9e05370b2`, `c30840c2e`.

## Quirks
- Spec violations load anyway (warn-only) → lenient by design; only a missing description hides a skill.
- Gating on reader tool is the only coupling between skills and toolset; a custom reader tool not named read/bash yields no skills section (unverified whether extensions can declare themselves readers).
- Skills section placed after project context, before cwd; adding a skill mid-session = section patch, not prefix rewrite.

## Durable variant (packages/durable)
- `pi-prompt` loads skills per cwd via `loadSkills({cwd, agentDir, skillPaths, includeDefaults:true})` (`experimental/durable/prompt.ts:34`); durable TUI lacks prompt templates (`experimental/durable/README.md:66`).

## Failures
- [[skills-hidden-when-read-tool-absent]]
- [[instruction-relative-paths-resolved-from-cwd]]
