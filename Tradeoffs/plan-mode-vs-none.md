---
type: tradeoff
concepts: [plan-mode, agent-profiles, tool-call-gate, minimal-default-toolset]
harnesses: [pi, opencode]
---
# plan-mode-vs-none

**Axis**: does the harness have a read-mostly "plan" state with an explicit switch to execution, or are plans just files?

| option | pi | opencode | evidence |
|---|---|---|---|
| Plans are files (PLAN.md), fresh session to implement | ✅ stated | possible, not the default | pi: "**No plan mode.** Gather context in one session, write plans to file, start fresh for implementation." (`3424550d2`) → [[no-plan-mode]] |
| Plan as an agent profile (permissions deny edits except the plan file) | example extension (tool-set swap + bash allowlist) | ✅ `plan` agent: `edit: {"*": deny, ".opencode/plans/*.md": allow}`, `task: {general: deny}` | pi: `examples/extensions/plan-mode/index.ts:22-23,168`; opencode: `packages/opencode/src/agent/agent.ts:156-180` → [[plan-mode]], [[agent-profiles]] |
| Exit = approval | — | ✅ `plan_exit` asks Yes/No, then a synthetic user message switches to `build` | `packages/opencode/src/tool/plan.ts:1-79` |
| Model may enter plan mode itself | ❌ | ❌ removed (`plan_enter` disabled `fa559b0385`) | → [[no-model-initiated-plan-entry]] |
| Prompt-only "READ-ONLY" rule | removed (`e3dd4f21d`, `b846a4bfc`) | description still says "Disallows all edit tools" while bash stays allowed | opencode: `packages/opencode/src/agent/agent.ts:158` |

**Known holes**
- opencode: plan could delegate edits to the `general` subagent (`b8ca71d309` 2026-05-09 fix; final form `3ad6923c61` denies `general` from plan). `explore` still has `bash: allow` (`packages/opencode/src/agent/agent.ts:205`) → [[read-only-mode-bypass-via-subagent]].
- pi: the example's bash allowlist is regex-based (`plan-mode/utils.ts:7,44`).

**When each wins**
- **No mode (pi)**: the plan must survive compaction and session switches verbatim (a file does), and the user is fine starting a fresh context for implementation. No mode state to leak through subagents or bash.
- **Plan agent (opencode)**: users who want a hard "don't touch files yet" guarantee for edit tools, with an approval checkpoint. Works only if every write path (bash, subagents) is covered by the same rules; prose claims are not enforcement.

Related: [[no-plan-mode]] · [[permission-prompts-vs-none]] · [[builtin-subagents-vs-none]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
