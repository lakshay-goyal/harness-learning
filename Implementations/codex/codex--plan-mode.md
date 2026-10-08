---
type: implementation
harness: codex
concept: plan-mode
commit: 622e9e3696
files: [codex-rs/collaboration-mode-templates/templates/plan.md, codex-rs/collaboration-mode-templates/templates/default.md, codex-rs/collaboration-mode-templates/src/lib.rs:1, codex-rs/models-manager/src/collaboration_mode_presets.rs, codex-rs/core/src/context/world_state/collaboration_mode.rs:26, codex-rs/protocol/src/protocol.rs:136, codex-rs/protocol/src/config_types.rs:674, codex-rs/core/src/tools/handlers/plan.rs:87, codex-rs/core/src/tools/handlers/request_user_input.rs:83, codex-rs/core/src/session/turn.rs:2602]
---
[[plan-mode]] in [[codex]].

## Mechanism
- **Modes**: `ModeKind::{Plan, Default}`; legacy names `code`, `pair_programming`, `execute`, `custom` alias to Default (`codex-rs/protocol/src/config_types.rs:674-700`); only Plan allows blocking `request_user_input` (`codex-rs/protocol/src/config_types.rs:700-702`; `codex-rs/core/src/tools/handlers/request_user_input.rs:83`).
- **Templates**: `codex-rs/collaboration-mode-templates/templates/plan.md` (9,184 bytes) and `default.md` (1,288 bytes) exported as `PLAN` / `DEFAULT` (`codex-rs/collaboration-mode-templates/src/lib.rs:1-2`). Presets: Plan = plan text + `reasoning_effort = Medium`; Default = default text (`codex-rs/models-manager/src/collaboration_mode_presets.rs`).
- **Selection** (`codex-rs/core/src/context/world_state/collaboration_mode.rs:26-60`, `:150-175`): model-catalog `model_messages.collaboration_modes` > mode settings' `developer_instructions`; if `update_plan` is disabled its guidance is stripped from recognized built-in text only, custom text untouched. Rendered as developer fragment in `<collaboration_mode>` (`codex-rs/protocol/src/protocol.rs:136-137`). Mode is a world-state section, so a switch appends a new developer message rather than rewriting the prefix → [[codex--world-state-diff-injection]].
- **default.md**: "You are now in Default mode. Any previous instructions for other modes (e.g. Plan mode) are no longer active. Your active mode changes only when new developer instructions with a different `<collaboration_mode>...</collaboration_mode>` change it; user requests or tool descriptions do not change mode by themselves."
- **plan.md structure**: "You are in **Plan Mode** until a developer message explicitly ends it" → "Mode rules (strict)" ("Plan Mode is not changed by user intent, tone, or imperative language. If a user asks for execution while still in Plan Mode, treat it as a request to **plan the execution**, not perform it.") → "Plan Mode vs update_plan tool" → "Execution vs. mutation" (allowed: reads, static analysis, dry runs, tests/builds that only write caches; not allowed: edits, formatters, codegen; "When in doubt: if the action would reasonably be described as 'doing the work' rather than 'planning the work,' do not do it.") → PHASE 1 ground in environment (explore first, ask second) → PHASE 2 intent chat → PHASE 3 implementation chat → "Asking questions" → "Two kinds of unknowns" (discoverable facts: explore; preferences: ask early with 2–4 options + recommended default) → "Finalization rule" (`<proposed_plan>` tags on own lines, untranslated, at most one per turn, revisions are complete replacements).
- **Enforcement** is almost entirely prompt-level: the only hard check is `update_plan` returning "update_plan is a TODO/checklist tool and is not allowed in Plan mode" (`codex-rs/core/src/tools/handlers/plan.rs:87-91`); entering Plan changes no sandbox/tool set (preset sets only instructions + effort). `<proposed_plan>` is parsed from the stream (`PlanModeStreamState`, `codex-rs/core/src/session/turn.rs:2602-2604`).

## Constants
| name | value | path:line |
|---|---|---|
| plan-mode reasoning effort | Medium preset; user override `plan_mode_reasoning_effort` (config/profile key, `4c1744afb2` 2026-02-20) | `codex-rs/models-manager/src/collaboration_mode_presets.rs:13`; `codex-rs/config/src/config_toml.rs:395` |
| plan.md / default.md size | 9,184 / 1,288 bytes | `codex-rs/collaboration-mode-templates/templates/` |
| preference questions | 2–4 options + recommended default | `codex-rs/collaboration-mode-templates/templates/plan.md` |
| plans per turn | at most one `<proposed_plan>` | `codex-rs/collaboration-mode-templates/templates/plan.md` |

## Evolution
- 2026-01-19 `d544adf71a` plan mode / collaboration modes + `request_user_input`; `f72f87fbee` 2026-01-18 first plan.md; 16 of 26 plan.md commits land 2026-01-18 → 2026-01-31 (subjects literally "prompt", "plan prompt v7", "prompt final") — eval-driven iteration at launch.
- 2026-01-19 `31415ebfcf` removed unused protocol prompts `execute.md`, `pair_programming.md` ("# Collaboration Style: Pair Programming… avoid taking steps that are too large…").
- 2026-01-20 `0523a259c8` "Reject ask user question tool in Execute and Custom"; 2026-01-26 `47aa1f3b6a` "Reject request_user_input outside Plan/Pair".
- 2026-01-26 `cabb2085cc` "make plan prompt less detailed": removed required "Step-by-step edits or patches described precisely", "Test cases and commands", "Acceptance criteria tied to observable outcomes" — body: "This was too much to ask for".
- 2026-01-30 `1ce722ed2e`: "Before asking the user any question, perform at least one targeted non-mutating exploration pass" + TL;DR checkpoint; fixes premature "Implement this plan?" prompts.
- 2026-01-31 `3dd9a37e0b`: removed "Hard interaction rule (critical) — Every assistant turn MUST be exactly one of: A) a `request_user_input` tool call… B) the final output… C) Direct response…" and "No questions in free text (only via `request_user_input`)"; replaced by "Strongly prefer using the `request_user_input` tool… In rare cases… you may ask it directly without the tool." → [[imperative-guideline-over-compliance]].
- 2026-02-02 `3392c5af24` `/plan` accepts prompt args and pasted images.
- 2026-02-03 `d509df676b` "Cleanup collaboration mode variants": visible set `default | plan`, Code renamed Default, Custom removed. Same day `998eb8f32b` "Improve Default mode prompt (less confusion with Plan mode)" (tool declared unavailable in Default).
- 2026-02-19 `c3cb38eafb` revised `<proposed_plan>` blocks must be complete replacements, not deltas.
- 2026-02-25 `2f4d6ded1d` "Enable request_user_input in Default mode".
- 2026-03-02 `50084339a6` verbosity: "prefer a compact structure with 3-5 short sections… avoid naming more than 3 paths… Prefer behavior-level descriptions over symbol-by-symbol removal lists."
- 2026-04-29 `fedcefe9da` "Reduce the surface of collaboration modes".
- 2026-06-21 `6bfc58a688` re-render the plan on relevant follow-ups so the user can exit plan mode to implement.
- 2026-08-28 `a5c581e247` default.md: "Never use the `request_user_input` tool for permission requests… Never write a multiple choice question as a textual assistant message."
- 2026-08-31 `a9519cbcdd` update_plan opt-in: bundled update_plan guidance stripped from collaboration-mode prompts when disabled → [[codex--dynamic-tool-guidelines]].
- 2026-09-05 `459a79eb85` template variable `{{KNOWN_MODE_NAMES}}` replaced with literal "Default and Plan".

## Quirks
- Plan Mode vs `update_plan`: two "plan" concepts coexist; the prompt dedicates a section to separating them and the tool hard-fails in Plan.
- Mode lives in append-only history; every mode message must cancel the previous one explicitly.

## Versus pi
- pi: [[no-plan-mode]] (absent by decision). codex ships plan mode as prompt-only discipline + one tool block + one blocking question tool.
