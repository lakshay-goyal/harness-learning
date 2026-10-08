---
type: implementation
harness: codex
concept: minimal-system-prompt
commit: 622e9e3696
files: [codex-rs/protocol/src/prompts/base_instructions/default.md, codex-rs/core/gpt_5_codex_prompt.md, codex-rs/prompts/src/update_plan_instructions.rs, codex-rs/prompts/src/permissions_instructions.rs:272]
---
[[minimal-system-prompt]] in [[codex]] — **contrast**: codex does not keep a tiny prompt; this note records its size and how it is kept in check.

## Mechanism
- Standing prompt per model: fallback 20,903 bytes / 275 lines (~5k tokens), catalog prompts 17,297–21,769 chars, codex-tuned 6,647–7,589 bytes. Long sections: planning with good/bad examples (`default.md:52-121`), preamble rules (`:29-50`), final-answer formatting spec (`:181-256`) → anatomy in [[codex--per-model-system-prompt]].
- Size discipline levers:
  - Model fit: models trained on the harness get the short prompt ("in-distribution models need less", `916fdc2a37`).
  - Strip guidance for disabled tools (update_plan off by default since `a9519cbcdd`) → [[codex--dynamic-tool-guidelines]].
  - Move volatile policy into appended developer fragments: `87f7226cca` (2026-01-12) deleted the ~35-line "Sandbox and approvals" section from `3a6a43ff5codex-rs/core/prompt.md` and all five gpt_* prompts, replaced by `<permissions instructions>` injected at session start and on change (`codex-rs/prompts/src/permissions_instructions.rs:272`) → [[permission-state-prompt]].
  - Delete harness constants restated in prose: `570eb5fe78` / `26d0d822a2` (2025-12-12) removed "Command line output will be truncated after 10 kilobytes or 256 lines…" → [[prompt-states-stale-harness-limits]].
  - Delete coding-only instructions from side prompts (memento compaction prompt → neutral handoff, `611e00c862`).

## Constants
| name | value | path:line |
|---|---|---|
| fallback prompt | 20,903 bytes / 275 lines | `codex-rs/protocol/src/prompts/base_instructions/default.md` |
| codex-tuned prompt | 6,647 bytes | `codex-rs/core/gpt_5_codex_prompt.md` |

## Evolution
- 2025-04-24 `31d0d7a305` 5,709 bytes → 2025-08-07 `81b148bda2` GPT-5 rewrite (+265/−75 lines) → 20.9 KB today. Growth by accretion of single-line behaviour patches (list in [[codex--per-model-system-prompt]]).
- 2025-08-12 `90d892f4fd` restored lines the rewrite dropped → [[prompt-rewrite-drops-load-bearing-lines]].
- 2025-11-19 `4985a7a444` anchor-splice of parallel-tool guidance silently failed → [[anchor-based-prompt-injection-silently-fails]].

## Versus pi
- [[pi--minimal-system-prompt]]: ~680 tokens, harness-owned, behaviour pushed into tools/docs. codex: ~5k tokens per model, vendor-tuned per model; it trades prompt size for behaviour control because it ships its own models. → [[single-vs-per-model-system-prompt]].
