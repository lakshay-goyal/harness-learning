---
type: implementation
harness: opencode
concept: patch-envelope-edit
commit: ecc4916b5a
files: [packages/opencode/src/tool/apply_patch.ts:47-52, packages/opencode/src/tool/apply_patch.ts:74, packages/opencode/src/tool/apply_patch.ts:206, packages/opencode/src/tool/apply_patch.ts:253-292, packages/opencode/src/patch/index.ts:353-381, packages/opencode/src/patch/index.ts:418, packages/opencode/src/patch/index.ts:460-483, packages/opencode/src/tool/registry.ts:297-300, packages/core/src/tool/apply-patch.ts:70-72, packages/opencode/src/tool/apply_patch.txt:1-33]
---
[[patch-envelope-edit]] in [[opencode]].

## Mechanism
### Legacy runtime (`packages/opencode`)
- Tool `apply_patch`, input `patchText` with `*** Begin Patch` / `*** Add File:` / `*** Update File:` / `*** Move to:` / `*** Delete File:` / `*** End Patch` (`packages/opencode/src/patch/index.ts:77-93` parses the headers).
- Exposed only when `usePatch` is true; `edit`/`write` are hidden then, so a request never carries both formats (`packages/opencode/src/tool/registry.ts:297-300`) → [[model-specific-toolset]].
- Hunk placement: `seekSequence` tries exact, then `trimEnd`, then `trim`, then `normalizeUnicode` (unicode punctuation → ASCII) on both sides (`packages/opencode/src/patch/index.ts:418,460-483`); `@@ change_context` lines seeked first (`:353`).
- Empty envelope → `patch rejected: empty patch`; no hunks → `apply_patch verification failed: no hunks found` (`packages/opencode/src/tool/apply_patch.ts:47-52`).
- Every target (and move destination) passes `assertExternalDirectoryEffect` (`apply_patch.ts:74,143`) → [[workspace-boundary-check]]; one `edit` permission ask for the whole patch (`:206`).
- Per file after write: `format.file` (`:253`) → [[post-edit-formatting]]; `lsp.touchFile` + diagnostics appended as "LSP errors detected in <rel>, please fix:" (`:269,292`) → [[lsp-diagnostics-feedback]].
- No per-path lock: `grep lock( apply_patch.ts` is empty, unlike `edit` → [[per-file-mutation-queue]].
- Model-facing description teaches the envelope by one worked example (Add + Update with `*** Move to:` + `@@` context + Delete) and two must-rules: always include an Add/Delete/Update header, prefix new-file lines with `+` (`packages/opencode/src/tool/apply_patch.txt:1-33`).
### v2 runtime (`packages/core`)
- Built-in `apply_patch` description: "All targets are resolved and approved before target contents are read. Operations apply sequentially; if a later operation fails, earlier operations remain applied…" (`packages/core/src/tool/apply-patch.ts:70-72`). Moves and atomic rollback are "deliberately unsupported in the first slice" (`specs/v2/schema-changelog.md:618`).

## Constants
| name | value | path:line |
|---|---|---|
| seek passes | 4 (exact, trimEnd, trim, unicode-normalized) | `packages/opencode/src/patch/index.ts:460-483` |
| model gate | `gpt-` && !`oss` && !`gpt-4` | `packages/opencode/src/tool/registry.ts:297-298` |

## Evolution
- 2026-01-17 `b7ad6bd839` "apply_patch tool for openai models" replaces the older `patch` tool.
- 2026-01-19 `dd0906be8c` description no longer says "This is a FREEFORM tool, so do not wrap the patch in JSON" (opencode exposes it as a JSON function tool).
- 2026-01-20 `9706aaf552` read-before-write (FileTime) assertions removed from the patch tool; `74bd52e8a7` emit edited events (LSP/snapshot refresh).
- 2026-03-17 `4b4dd2b882` added to the `EDIT_TOOLS` permission filter → [[alternate-edit-tool-skips-edit-pipeline]].
- 2026-08-31 `f7da00f35e` omit empty move path.

## Quirks / drift
- `gpt.txt:27` "Always use apply_patch for manual code edits" is also routed to `gpt-oss` ids, which never get the tool; `codex.txt:12` says "Use Read to view files, Edit to modify files" though Edit is hidden for those ids → [[prompt-names-unavailable-tools]].
- `write` and `apply_patch` share no lock with `edit`.

pi rejects this format outright (Codex bridge "APPLY_PATCH DOES NOT EXIST") → [[foreign-harness-tool-hallucination]]; there is no pi implementation note.
