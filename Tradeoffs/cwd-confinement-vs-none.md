---
type: tradeoff
concepts: [workspace-boundary-check, path-normalization, tool-only-isolation, permission-ruleset]
harnesses: [pi, opencode]
---
# cwd-confinement-vs-none

**Axis**: do file tools treat paths outside the project specially, or is the whole filesystem one space?

| option | pi | opencode | evidence |
|---|---|---|---|
| No boundary; cwd is only the default base | ✅ `resolvePath` accepts any absolute path or `../`; `isInsideCwd` used for display only | — | pi: `packages/coding-agent/src/utils/paths.ts:103-118`; "does not prevent commands from accessing other paths" (`packages/coding-agent/docs/security.md:19`) → [[no-cwd-confinement]] |
| Soft boundary: separate permission for outside paths | example `protected-paths.ts` (denylist, not confinement) | ✅ `external_directory` ask on read/edit/write/apply_patch/grep/glob and shell path args; default `*: ask` | opencode: `packages/opencode/src/tool/external-directory.ts:13-44`; `packages/opencode/src/agent/agent.ts:122-123` → [[workspace-boundary-check]] |
| Containment check | — | legacy: lexical `path.relative` (in-repo symlink to outside counts as inside); v2: `fs.realPath` + revalidate right before write | `packages/core/src/location-mutation.ts:84-103`; `specs/v2/schema-changelog.md:261-270` → [[read-path-traversal]] |
| Bash | unconfined (`cd` anywhere) | unconfined at OS level; path args parsed for the ask only | both → [[no-sandbox]] |
| Read-only reference dirs | — | config `references` whitelisted for read; v2 references read-only | `packages/opencode/src/agent/agent.ts:102-112` → [[project-references]] |

**When each wins**
- **None (pi)**: honest about the real boundary (the OS user); path quirks can be normalized instead of rejected; zero prompts for monorepo siblings or `~/.config`.
- **Soft boundary (opencode)**: catches the common model mistake (editing the wrong checkout, writing to `/tmp` of another project) with one prompt. It is a speed bump, not a jail: bash bypasses it, and v1's lexical check misses symlinks. Only meaningful paired with approvals ([[permission-prompts-vs-none]]).

Related: [[no-cwd-confinement]] · [[permission-prompts-vs-none]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
