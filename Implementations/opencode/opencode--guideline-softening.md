---
type: implementation
harness: opencode
concept: guideline-softening
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt/anthropic.txt, packages/opencode/src/session/prompt/default.txt:68, packages/opencode/src/session/prompt/default.txt:84-86, packages/opencode/src/session/prompt/codex.txt:43-49, packages/opencode/src/session/prompt/plan.txt, packages/opencode/src/command/template/initialize.txt, .opencode/command/rmslop.md:1-15]
---
[[guideline-softening]] in [[opencode]]. Both directions are visible: softening in family prompts, escalation in mode reminders.

## Mechanism
- **Softened / removed after over-compliance**:
  - Brevity: "You MUST answer concisely with fewer than 4 lines" (`dac1506680` 2025-08-11) → "matching the level of detail … with the level of complexity of the user's query" (`5a507023a6` 2025-09-30) → [[imperative-guideline-over-compliance]]. The fallback `default.txt:84` still has "fewer than 4 lines".
  - Safety boilerplate ("Refuse to write code or explain code that may be used maliciously…") removed from anthropic.txt (`dac1506680`) and from the fallback (`8ee939c741` 2026-03-18); motive unstated (over-refusal likely, unverified). Residue: half-sentence "IMPORTANT: Before you begin work, think about what the code you're editing is supposed to do…" (`default.txt:86`).
  - GPT autonomy push ("persist end-to-end") replaced by "Do not infer authorization for work beyond the user's request" on origin/v2 (`a7b8174917` 2026-09-02) → [[autonomy-prompt-overreach]].
  - Codex "Ask only when needed" → explicit enumerated reasons to ask (`packages/opencode/src/session/prompt/codex.txt:43-49`, `5622c53e1f`) → [[excessive-permission-questions]].
  - `/init` "Make it about 150 lines long" → "Would an agent likely miss this without help? If not, leave it out" (`897d83c589`).
- **Escalated instead**: plan-mode reminder grew from "you MUST NOT make any edits" (`a826936702`) to "CRITICAL: Plan mode ACTIVE … STRICTLY FORBIDDEN … ZERO exceptions" (`ca3769b7fa`) because bash stayed allowed by permissions → [[plan-mode]].
- **Re-hardened after softening**: "IMPORTANT: Always use the TodoWrite tool" dropped in `795b845782` (2025-10-25), re-added 3 days later `22821744ef` → [[todo-tool-usage-calibration]].
- Team's own anti-pattern list as a cleanup command instead of a prompt rule: `.opencode/command/rmslop.md` (extra comments, defensive try/catch, `any` casts, emoji) → [[over-commenting-code]].

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- See the dated commits above; most land in the per-family prompt files → [[opencode--per-model-system-prompt]].

## Quirks / drift
- Softening is applied per prompt file, so the same rule survives in some families (default.txt "fewer than 4 lines", "DO NOT ADD ***ANY*** COMMENTS" at `:68`) after removal in others.

Contrast: pi softens one shared prompt (capability statements, named anti-patterns) → [[pi--guideline-softening|pi]].
