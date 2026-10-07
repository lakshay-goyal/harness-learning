---
type: failure
concepts: [tool-description-design, provider-identity-shim]
harnesses: [pi, opencode]
---
**Symptom** — Codex-trained models running inside pi called tools from their native harness that pi does not have: `apply_patch`/`applyPatch` for file edits, `update_plan`/`read_plan`/`todowrite`/`todoread` for planning.

**Root cause** — Models carry the tool vocabulary of the harness they were trained in; the upstream Codex instructions shipped with the provider even described those tools.

**Fix · [[pi]]**
- `1650041a6` 2026-01-04 — "bridge" prompt `pi-codex-bridge.ts` after the upstream Codex instructions: `<critical_rule priority="0">` "❌ APPLY_PATCH DOES NOT EXIST → ✅ USE "edit" INSTEAD — NEVER use: apply_patch, applyPatch", "❌ UPDATE_PLAN DOES NOT EXIST …", plus a verification checklist ("Using edit, not apply_patch; No plan tools used; Only the tools listed above are called").
- `bb50738f7` 2026-01-05 — pi system prompt appended after the bridge in one message.
- `6484ae279` 2026-01-16 (#737) — bridge + upstream prompts deleted, replaced by allowlisted static instructions; `4068bc556` 2026-01-17 — Codex uses pi's normal system prompt as `instructions` (no bridge since).

**Lesson** — Models bring their training harness's tool names; either alias those tools to yours or state explicitly that they don't exist — and prefer the former over shouting rules. (unverified whether symptom recurred after the bridge was dropped.)

Related: [[tool-description-design]] · [[provider-identity-shim]] · [[search-replace-edit]] · [[no-todo-tool]] · [[harness-identity]]

**Fix · [[pi]]** (prompting angle, appended by 04-prompting writer) — `6dcb64565` 2026-01-10 "Prepare for alternative Codex harness certification" sits between bridge and allowlisted-instructions phases; the durable outcome is [[harness-identity]]: preamble "operating inside pi, a coding agent harness" (`4068bc556`, HEAD `packages/coding-agent/src/core/system-prompt.ts:155-156`) instead of tool-by-tool `critical_rule` denials. Full Codex prompt detour timeline in [[pi--harness-identity]].

**Fix · [[opencode]]** — opposite strategy: give the model its native tools. `b7ad6bd839` 2026-01-17 `apply_patch` for `gpt-*` ids instead of edit/write (`packages/opencode/src/tool/registry.ts:297-300`) → [[model-specific-toolset]], [[patch-envelope-edit]]. Case-mismatched names (`Read`) auto-repaired by lowercasing in `experimental_repairToolCall`; anything else becomes the hidden `invalid` tool's error result (`packages/opencode/src/session/llm.ts:296-317`, `0a42068fbb` 2025-08-04). See [[opencode--tool-argument-repair]].
