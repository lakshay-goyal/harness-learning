---
type: implementation
harness: opencode
concept: post-edit-formatting
commit: ecc4916b5a
files: [packages/opencode/src/format/index.ts:118-140, packages/opencode/src/format/formatter.ts, packages/opencode/src/tool/edit.ts:155-158, packages/opencode/src/tool/write.ts:65, packages/opencode/src/tool/apply_patch.ts:253]
---
[[post-edit-formatting]] in [[opencode]].

## Mechanism
- `Format.Service.file(path)` runs the matching formatter and returns whether one ran (`packages/opencode/src/format/index.ts`).
- `edit`: write → `format.file` → if formatted, `contentNew = Bom.syncFile(...)` re-reads with the original BOM; the returned diff is recomputed from the post-format content (`packages/opencode/src/tool/edit.ts:155-170`).
- `write` (`packages/opencode/src/tool/write.ts:65`) and `apply_patch` (`packages/opencode/src/tool/apply_patch.ts:253`) call the same hook.
- Built-in table: 26 formatter `Info` entries (prettier, biome, oxfmt, gofmt, ruff, uv, clang-format, ktlint, rubocop, shfmt, terraform, dart, gleam, …) in `packages/opencode/src/format/formatter.ts`.
- Config: `formatter` unset/false → "all formatters are disabled" (`packages/opencode/src/format/index.ts:120-127`); `true` = built-ins; object = per-name overrides/disables (`:132-137`).
- Order inside `edit`: permission ask (diff) → write → format → events → diagnostics → [[lsp-diagnostics-feedback]].
- v2 runtime: no formatter yet — "TODO: Add formatter integration after V2 formatter runtime exists." (`packages/core/src/tool/edit.ts:85`; `packages/core/src/tool/write.ts:42`).

## Constants
| name | value | path:line |
|---|---|---|
| default | disabled unless `formatter` set | `packages/opencode/src/format/index.ts:120` |

## Evolution
- 2025-11-18 `81ebf56cf1` top-level `formatter: false` added (on by default before).
- 2026-04-16 `220e3e9a2b` "refactor: make formatter config opt-in (#22997)". No rationale in body; rewriting user files outside the edited span is a plausible reason (unverified).
- 2026-05-03 `6b68b1020e` docs clarify opt-in.

## Quirks / drift
- The permission prompt shows the pre-format diff; the user approves bytes that the formatter then changes.

pi has no formatter hook; formatting is left to the model's shell usage.
