---
type: failure
concepts: [tool-description-design, provider-identity-shim, per-model-system-prompt]
harnesses: [pi, codex]
---
**Symptom** — Codex-trained models running inside pi called tools from their native harness that pi does not have: `apply_patch`/`applyPatch` for file edits, `update_plan`/`read_plan`/`todowrite`/`todoread` for planning.

**Root cause** — Models carry the tool vocabulary of the harness they were trained in; the upstream Codex instructions shipped with the provider even described those tools.

**Fix · [[pi]]**
- `1650041a6` 2026-01-04 — "bridge" prompt `pi-codex-bridge.ts` after the upstream Codex instructions: `<critical_rule priority="0">` "❌ APPLY_PATCH DOES NOT EXIST → ✅ USE "edit" INSTEAD — NEVER use: apply_patch, applyPatch", "❌ UPDATE_PLAN DOES NOT EXIST …", plus a verification checklist ("Using edit, not apply_patch; No plan tools used; Only the tools listed above are called").
- `bb50738f7` 2026-01-05 — pi system prompt appended after the bridge in one message.
- `6484ae279` 2026-01-16 (#737) — bridge + upstream prompts deleted, replaced by allowlisted static instructions; `4068bc556` 2026-01-17 — Codex uses pi's normal system prompt as `instructions` (no bridge since).

**Fix · [[codex]]** (the owning harness's side)
- Symptom: codex's own models invoked misspelled tool names (`applypatch`, `apply-patch`) and emitted Codex-web / ChatGPT citation syntax ("【F:README.md†L5-L14】") the CLI cannot render; the very first Rust prompt was the Codex-web container prompt ("internet access is disabled in the container", "You do not need to `git commit` your changes; this will be done automatically for you.", `31d0d7a305:codex-rs/core/prompt.md`).
- `d31e149cb1` 2025-08-05 / `81b148bda2` 2025-08-07 — container text removed; "Use the `apply_patch` tool to edit files (NEVER try `applypatch` or `apply-patch`, only `apply_patch`)" and "NEVER output inline citations like "【F:README.md†L5-L14】" … The CLI is not able to render these" (`codex-rs/protocol/src/prompts/base_instructions/default.md:132,147`).
- `5f8984aa7d` 2025-08-11 — harness also **aliases** `applypatch` instead of only forbidding it ([[tool-name-training-artifact]]).
- `update_plan` made opt-in `a9519cbcdd` 2026-08-31: codex's own trained tool now also "doesn't exist" by default ([[plan-checklist-tool]]).

**Lesson** — Models bring the tool names and output conventions of every product they were trained for: alias the tools you can (prefer this over shouting rules), and on each surface explicitly forbid the other surfaces' conventions.

Related: [[tool-description-design]] · [[provider-identity-shim]] · [[search-replace-edit]] · [[no-todo-tool]] · [[harness-identity]] · [[per-model-system-prompt]] · [[patch-envelope-edit]] · [[plan-checklist-tool]] · [[codex--tool-description-design|codex]]

**Fix · [[pi]]** (prompting angle, appended by 04-prompting writer) — `6dcb64565` 2026-01-10 "Prepare for alternative Codex harness certification" sits between bridge and allowlisted-instructions phases; the durable outcome is [[harness-identity]]: preamble "operating inside pi, a coding agent harness" (`4068bc556`, HEAD `packages/coding-agent/src/core/system-prompt.ts:155-156`) instead of tool-by-tool `critical_rule` denials. Full Codex prompt detour timeline in [[pi--harness-identity]].
