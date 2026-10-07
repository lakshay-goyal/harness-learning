---
type: failure
concepts: [plan-mode, message-role-layering, dynamic-tool-guidelines, minimal-system-prompt]
harnesses: [codex]
---
**Symptom** — The model confused Default and Plan modes: treated a user's imperative ("just do it") as leaving Plan mode, used `update_plan` as if it were Plan mode, called `request_user_input` outside Plan, produced partial/delta plans on revision, asked questions before exploring, produced over-detailed plans.

**Root cause** — Mode lives in an append-only developer message; earlier mode text stays in history, user text sounds authoritative, and tool availability per mode was gated only in the prompt.

**Fix · [[codex]]**
- plan.md "Plan Mode is not changed by user intent, tone, or imperative language. If a user asks for execution while still in Plan Mode, treat it as a request to **plan the execution**" + hard error "update_plan is a TODO/checklist tool and is not allowed in Plan mode" (`codex-rs/core/src/tools/handlers/plan.rs:87-91`).
- Runtime gating of the question tool: `0523a259c8` 2026-01-20 "Reject ask user question tool in Execute and Custom", `47aa1f3b6a` 2026-01-26 "Reject request_user_input outside Plan/Pair"; later reversed `2f4d6ded1d` 2026-02-25 "Enable request_user_input in Default mode".
- `998eb8f32b` 2026-02-03 "Improve Default mode prompt (less confusion with Plan mode)": "Any previous instructions for other modes (e.g. Plan mode) are no longer active."; default.md "user requests or tool descriptions do not change mode by themselves" (`codex-rs/collaboration-mode-templates/templates/default.md`).
- `c3cb38eafb` 2026-02-19 revised plans are complete replacements; `1ce722ed2e` 2026-01-30 explore before the first question; `cabb2085cc` 2026-01-26 less detail ("This was too much to ask for").

**Lesson** — When mode lives in append-only developer messages, every mode message must explicitly cancel the previous mode and state who may change it; when tool availability depends on mode, gate it at runtime *and* keep the description in sync — prompt-only gating leaks.

Related: [[plan-mode]] · [[message-role-layering]] · [[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[structured-user-question-tool]] · [[codex--plan-mode|codex]]
