---
type: implementation
harness: opencode
concept: project-references
commit: ecc4916b5a
files: [packages/core/src/reference.ts:58-105, packages/opencode/src/session/system.ts:71-104, packages/opencode/src/agent/agent.ts:102-112, packages/core/src/reference/guidance.ts:14-61]
---
[[project-references]] in [[opencode]].

## Mechanism
- **Sources** (`packages/core/src/reference.ts:58-105`): config `references` (or `reference`) maps names to a local path or a remote git repository (+ optional branch). Git sources resolve to `Repository.cachePath(global.repos, repo, branch)` and are fetched in the background (`cache.ensure({refresh: true})`, failures only logged); invalid branches/non-remote repos are skipped. Each has optional `description` and `hidden`.

### Legacy runtime
- `SystemPrompt.environment` appends, after `<env>`, "Project references provide additional directories that can be accessed when relevant." + `<available_references>` with `<reference><name><path><description>` — only references **with a description**, sorted by name (`packages/opencode/src/session/system.ts:71-104`). Rebuilt into the system prompt every step.
- Permissions: every reference dir glob is added to the agent whitelist (with the truncation dir, tmp and skill dirs) so out-of-workspace reads are allowed (`packages/opencode/src/agent/agent.ts:102-112`) → [[workspace-boundary-check]].

### v2 runtime
- `ReferenceGuidance` is a Context Source `core/reference-guidance` (`packages/core/src/reference/guidance.ts:50-61`): baseline = same block; update = "The available project references have changed. This list supersedes the previous reference list." + new block; removal = "Project reference guidance is no longer available. Do not use previously listed references." Changes append as a mid-conversation system message instead of rewriting the prefix ([[opencode--transcript-carried-system-prompt]]).

## Constants
| name | value | path:line |
|---|---|---|
| listing filter | description required | `packages/opencode/src/session/system.ts:72`; `packages/core/src/reference/guidance.ts:42` |

## Evolution
- 2026-05-09 `40d5ea1cf1` `scout` agent for repo research; removed 2026-06-02 `a639fe7a08` (reason not stated — unverified).
- 2026-06-09 `6566ede935` consolidate references; 2026-06-10 `8a2cfc00c9` (#31601) add project reference guidance.

## Quirks / drift
- Legacy puts the reference list inside the per-step system string, so editing references mid-session rewrites the cached prefix; v2 fixes that by design.
- A reference without a description is still readable (whitelisted) but invisible to the model.

Contrast: pi has no reference mechanism; extra directories reach the model only via context files or the user.
