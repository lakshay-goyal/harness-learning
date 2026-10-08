---
type: implementation
harness: opencode
concept: search-replace-edit
commit: ecc4916b5a
files: [packages/opencode/src/tool/edit.ts:30-45, packages/opencode/src/tool/edit.ts:80-82, packages/opencode/src/tool/edit.ts:140-199, packages/opencode/src/tool/edit.ts:682-728, packages/opencode/src/tool/edit.txt:4, packages/opencode/src/tool/write.ts:41-89, packages/opencode/src/tool/write.txt:5, packages/core/src/tool/edit.ts:59-117, packages/core/src/file-mutation.ts:32-63]
---
[[search-replace-edit]] in [[opencode]].

## Mechanism
### Legacy runtime
- `edit(filePath, oldString, newString, replaceAll?)`. Relative paths joined to `instance.directory` (`packages/opencode/src/tool/edit.ts:80-82`).
- `replace()` guards: identical strings → "No changes to apply"; empty `oldString` on an existing file → error ("use write for an intentional full-file replacement") (`edit.ts:682-692`).
- Matching cascade of 9 replacers; first candidate that occurs exactly once wins; `replaceAll` replaces the first candidate found everywhere (`edit.ts:695-721`) → [[fuzzy-edit-matching]].
- Distinct errors per remedy: "Could not find oldString in the file. It must match exactly…" vs "Found multiple matches for oldString. Provide more surrounding context…" (`edit.ts:723-728`) → [[ambiguous-tool-error-causes-retry-loop]].
- CRLF normalized for matching, original ending restored (`convertToLineEnding`, `edit.ts:30-33`); BOM kept via `Bom.join`/`Bom.syncFile`.
- Whole read-modify-write under a per-resolved-path `Semaphore(1)` (`edit.ts:35-45`) → [[per-file-mutation-queue]].
- Sequence: diff → `ctx.ask({permission:"edit", metadata:{diff}})` → write → format + re-read → events → "Edit applied successfully." + LSP errors (`edit.ts:140-199`) → [[post-edit-formatting]], [[lsp-diagnostics-feedback]].
- `write`: absolute/relative path, external-directory check, format, "Wrote file successfully." + diagnostics (`packages/opencode/src/tool/write.ts:41-89`); no lock.
- `edit`, `write`, `apply_patch` share the single `edit` permission (`packages/opencode/src/permission/index.ts:204-219`) → [[permission-ruleset]].
- No read-before-write check since `76a141090e` (2026-04-16).
### v2 runtime
- Exact `indexOf` counting only; >1 match without `replaceAll` is an error (`packages/core/src/tool/edit.ts:59`); TODO "Port V1 fuzzy correction strategies only after exact-edit behavior is established" (`:84`).
- Conditional write: bytes read at approval time compared at commit (`FileMutation.writeIfUnchanged`, `StaleContentError`, `packages/core/src/file-mutation.ts:32,61-63`); surfaced as "File changed after permission approval. Read it again before editing." (`packages/core/src/tool/edit.ts:115`).

## Constants
| name | value | path:line |
|---|---|---|
| replacers in cascade | 9 | `packages/opencode/src/tool/edit.ts:695-705` |

## Evolution
- 2025-06-05 `35b03e4cb3` `multiedit` commented out of the registry; 2026-04-21 `2486621ca1` deleted ("kill unused tool"). It was "multiple find-and-replace operations" on one file (`2486621ca1^:packages/opencode/src/tool/multiedit.txt:1`).
- 2025-09-05 `900fe5ca04` split "not found" vs "multiple" errors to stop a "doom editing loop".
- 2025-12-15 `5cf126d489` per-file lock; 2026-04-20 `8bc4f91fd9` parallel edits overriding each other → [[concurrent-file-mutation-interleave]].
- 2026-01-12 `66f9bdab32` explicit success line → [[empty-success-output-read-as-failure]].
- 2026-02-12 `624dd94b5d` current error strings ("tool outputs to be more llm friendly").
- 2026-04-16 `76a141090e` FileTime module deleted: it had thrown "You must read file X before overwriting it. Use the Read tool first" and an mtime/size staleness error (`76a141090e^:packages/opencode/src/file/time.ts:92-99`). Commit has no body; rationale unverified.
- 2026-06-05 `236cfcbbc3` empty `oldString` rejected; disproportionate-match veto → [[fuzzy-edit-rewrites-untouched-lines]].

## Quirks / drift
- `edit.txt:4` still says "This tool will error if you attempt an edit without reading the file"; `write.txt:5` "This tool will fail if you did not read the file first" — both false since `76a141090e` → [[tool-description-drifts-from-implementation]].
- `edit.txt` quotes error strings the code no longer emits (prompts.md finding; strings changed in `624dd94b5d`).
- GPT ids never see `edit`/`write` → [[patch-envelope-edit]].

Contrast: pi uses a multi-edit array against the original and never had a read-before-edit guard → [[pi--search-replace-edit|pi]].
