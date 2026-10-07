---
type: implementation
harness: pi
concept: search-replace-edit
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/edit.ts:21, packages/coding-agent/src/core/tools/edit.ts:103, packages/coding-agent/src/core/tools/edit.ts:143, packages/coding-agent/src/core/tools/edit-diff.ts:300, packages/coding-agent/src/core/tools/edit-diff.ts:376, packages/coding-agent/src/core/tools/write.ts:11, packages/coding-agent/src/core/tools/write.ts:52, packages/durable/src/tools/edit.ts:45]
---
[[search-replace-edit]] in [[pi]].

Two file-mutating built-ins: `edit` (exact old→new replacement, multi-edit) and `write` (whole-file create/overwrite). Both default-active (`DEFAULT_TOOL_NAMES`, see [[pi--minimal-default-toolset|minimal-default-toolset]]), both run inside the [[per-file-mutation-queue]], both `constrainedSampling: {type:"json_schema", strict:"prefer"}` ([[constrained-tool-sampling]]; `edit.ts:156`, `write.ts:57`).

## Mechanism

### `edit` schema (`packages/coding-agent/src/core/tools/edit.ts:21-41`)
| param | type | description (verbatim) |
|---|---|---|
| `path` | string | "Path to the file to edit (relative or absolute)" |
| `edits` | array of `{oldText,newText}` | "One or more targeted replacements. Each edit is matched against the original file, not incrementally. Do not include overlapping or nested edits. If two changes touch the same block or nearby lines, merge them into one edit instead." |
| `edits[].oldText` | string | "Exact text for one targeted replacement. It must be unique in the original file and must not overlap with any other edits[].oldText in the same call." |
| `edits[].newText` | string | "Replacement text for this targeted edit." |
- Both object schemas take `{}` options → **additionalProperties allowed** (`edit.ts:30,41`; `a1b336d73` #6278: model-invented extra fields were rejecting valid edits).
- `edits` min length 1 enforced at runtime, not in schema: "Edit tool input is invalid. edits must contain at least one replacement." (`edit.ts:136-141`).
- Description (`edit.ts:151-152`): "Edit a single file using exact text replacement. Every edits[].oldText must match a unique, non-overlapping region of the original file. If two changes affect the same block or nearby lines, merge them into one edit instead of emitting overlapping edits. Do not include large unchanged regions just to connect distant changes."
- `promptSnippet` "Make precise file edits with exact text replacement, including multiple disjoint edits in one call"; 4 `promptGuidelines` (`edit.ts:43-51`) → rendered in `<rules>` ([[dynamic-tool-guidelines]]):
  1. "Use edit for precise changes (edits[].oldText must match exactly)"
  2. "When changing multiple separate locations in one file, use one edit call with multiple entries in edits[] instead of multiple edit calls"
  3. "Each edits[].oldText is matched against the original file, not after earlier edits are applied. Do not emit overlapping or nested edits. Merge nearby changes into one edit."
  4. "Keep edits[].oldText as small as possible while still being unique in the file. Do not pad with large unchanged regions."
- `prepareArguments: prepareEditArguments` (`edit.ts:103-134,158`) runs before schema validation — repairs `edits` as JSON string, single object, legacy top-level `oldText/newText`. Detail in [[pi--tool-argument-repair|tool-argument-repair]].
- `renderShell: "self"` (`edit.ts:157`).

### `edit` execution (`edit.ts:159-215`)
1. `absolutePath = resolveToCwd(path, ctx?.cwd || cwd)` (`edit.ts:161`; [[path-normalization]]; `62835ea81` #8627 session cwd). No cwd confinement ([[no-cwd-confinement]]).
2. Enter `withFileMutationQueue(absolutePath, …)` (`edit.ts:163`).
3. `ops.access` → on failure `Could not edit file: {path}. Error code: ENOENT.` (errno classified; `edit.ts:174-182`, `43ee9b77e`/`ebdf3cf45`, commit subject #3955).
4. `ops.readFile` → `toString("utf-8")` (`edit.ts:185-186`).
5. `splitBom` — "Strip BOM before matching. The model will not include an invisible BOM in oldText." (`edit.ts:190-191`; `d9adf659c`). `detectLineEnding` = whichever of first `\r\n` vs first `\n` occurs first (`edit-diff.ts:11-17`); `normalizeToLF` also converts lone `\r` (`edit-diff.ts:19-21`; `8c43a9fbc`/`c214a3340` #355/#360).
6. `applyEditsToNormalizedContent(normalizedContent, edits, path)` (`edit-diff.ts:300-362`):
   - each edit's oldText/newText → LF (`:305-308`); empty oldText → `oldText must not be empty in {path}.` / `edits[i].oldText must not be empty…` (`:275-280,310-314`).
   - `fuzzyFindText` per edit: exact `indexOf` first, fuzzy normalized fallback (`:207-245`) → see [[pi--fuzzy-edit-matching|fuzzy-edit-matching]]. If any edit needed fuzzy, all edits are re-matched against the fuzzy-normalized whole file (`:316-318`).
   - not found → single: "Could not find the exact text in {path}. The old text must match exactly including all whitespace and newlines."; multi: "Could not find edits[i] in {path}. The oldText must match exactly including all whitespace and newlines." (`:253-262`).
   - uniqueness via `countOccurrences` (always in fuzzy-normalized space, `:247-251`) → "Found N occurrences of the text in {path}. The text must be unique. Please provide more context to make it unique." / "Found N occurrences of edits[i] in {path}. Each oldText must be unique. Please provide more context to make it unique." (`:264-273`).
   - sort by offset; overlap → "edits[i] and edits[j] overlap in {path}. Merge them into one edit or target disjoint regions." (`:341-350`).
   - apply: exact → `applyReplacements` reverse-order splice (`:111-120`); fuzzy → `applyReplacementsPreservingUnchangedLines` (`:132-173`).
   - identical result → "No changes made to {path}. The replacement produced identical content. This might indicate an issue with special characters or the text not existing as expected." (multi: "…The replacements produced identical content.") (`:282-289,357-359`; text since `52325adb9`).
7. Write `bom + restoreLineEndings(newContent, originalEnding)` (`edit.ts:197-198`) → mixed-ending files become uniform in the detected ending.
8. Result text `Successfully replaced N block(s) in {path}.` (`edit.ts:204-209`); `details: {diff, patch, firstChangedLine}` (`:210`) — diff/patch are **not** model-visible (TUI/extensions only; `60a55a239` added `patch`).

### Diff generation (`edit-diff.ts`)
- `generateDiffString(old,new)` (`:376-499`): `diff.diffLines`, display lines `+NN`/`-NN`/` NN` with padded line numbers, 4 context lines, long unchanged gaps collapsed to `...` (`a0734bd16`), returns `firstChangedLine` (new-file numbering).
- `generateUnifiedPatch(path, old, new, contextLines=4)` (`:365-370`): `Diff.createTwoFilesPatch`, headers only.
- `computeEditsDiff` (`:514-543`, uses `resolveToCwd` `:519`): preview diff computed **before execution** so permission prompts / pending TUI rows show it (`1ed009e2c` #393; renderer `renderers/edit.ts`); exported for extensions (`2b46f3886` #5756); preview stabilized `f7cd613ee` #3134.

### Abort discipline (edit + write)
- `edit.ts:164-171`, `write.ts:68-74`: "Do not reject from an abort event listener here: that would release the mutation queue while an in-flight filesystem operation may still finish. Checking signal.aborted after each await observes the same aborts while keeping the queue locked until the current operation has settled." → `throwIfAborted()` after every await, error "Operation aborted".

### `write` (`packages/coding-agent/src/core/tools/write.ts`)
- Schema (`:11-14`): `path` (string), `content` (string).
- Description (`:52-53`, unchanged since `ffc9be886` 2025-10-17): "Write content to a file. Creates the file if it doesn't exist, overwrites if it does. Automatically creates parent directories."
- Snippet "Create or overwrite files"; guideline "Use write only for new files or complete rewrites." (`:16-19`).
- Execution (`:64-89`): `resolveToCwd(path, ctx?.cwd || cwd)` → queue → `ops.mkdir(dirname)` (mkdir -p) → `ops.writeFile(absolutePath, content)` (utf-8) → `Successfully wrote to {path}` (`:86`), `details: undefined`.
- No BOM/CRLF preservation on overwrite, no backup, no atomic temp+rename, no read-before-write check (observed absence, `write.ts:67-89`).

### Pluggable I/O
- `EditOperations {access, readFile, writeFile}` / `WriteOperations {mkdir, writeFile}` default to node fs; replaceable for SSH/VM ([[pluggable-tool-backends]], `9ed88646a`).

### What is deliberately absent
- No read-before-edit tracking / staleness check — nothing records reads; `getFileRevision` only used by models-store/auth-storage (observed absence). Prompt rule "Use read to examine files before editing" was dropped in `235b247f1` (see [[pi--tool-description-design|tool-description-design]]).
- No line-number anchors, no "replace all" flag, no indentation-insensitive fuzzy (observed absence).
- Not a lock vs `bash` (a `sed -i` in bash can race an edit).

## Constants
| name | value | path:line |
|---|---|---|
| diff context lines | 4 | `packages/coding-agent/src/core/tools/edit-diff.ts:365` |
| edits min length | 1 (runtime) | `packages/coding-agent/src/core/tools/edit.ts:136-141` |
| constrainedSampling | `{type:"json_schema", strict:"prefer"}` | `edit.ts:156`, `write.ts:57` |
| `WRITE_PARTIAL_FULL_HIGHLIGHT_LINES` (renderer) | 50 | `packages/coding-agent/src/core/tools/renderers/write.ts:29` |

## Evolution
- 2025-10-17 `ffc9be886`: v0 edit `{path, oldText, newText}`, desc "Edit a file by replacing exact text. The oldText must match exactly (including whitespace). Use this for precise, surgical edits."; snippet "Make surgical edits to files (find exact text and replace)". Write desc already final.
- 2025-11-11/12 `159075cad` → `9e3e319f1`: not-found / "Found N occurrences… Please provide more context to make it unique." error texts; tools throw (loop renders) — see [[pi--tool-error-as-result|tool-error-as-result]].
- 2025-11-24 `52325adb9` (v0.9.2): "No changes made… identical content" error.
- 2025-12-29/30 `c214a3340` (#360) / `8c43a9fbc`: CRLF normalize + restore.
- 2026-01-01 `1ed009e2c` (#393): diff shown before execution.
- 2026-01-02 `d9adf659c`: BOM strip/restore.
- 2026-01-14 `0c135d014` (#713): fuzzy fallback ([[fuzzy-edit-matching]]).
- 2026-03-20 `74a46fc7e` (#2327): edit+write wrapped in file mutation queue.
- 2026-03-22 `235b247f1`: built-in tools become ordinary `ToolDefinition`s with own snippets/guidelines + split renderers.
- 2026-03-27 `20a57e759`: multi-edit; **dual-mode** schema (`oldText/newText` OR `edits[]`) + 5 guidelines; `a0734bd16` diff gap collapse.
- 2026-03-28 `e773527b3` (#2639): `edits[]` only — CHANGELOG: "eliminating the mixed single-edit and multi-edit modes that caused repeated invalid tool calls and retries" → [[edit-tool-dual-mode-confusion]].
- 2026-03-29 `b5f425ad1`: `prepareArguments` folds legacy `oldText/newText` (resumed sessions).
- 2026-04-15 `f7cd613ee` (#3134) stable previews; 2026-04-18 `a2ec01e12` (#3370) stringified `edits`; 2026-04-30 `43ee9b77e`/`ebdf3cf45` errno-classified access errors.
- 2026-05-21 `60a55a239`: unified `patch` in details; 2026-06-18 `2b46f3886` (#5756) edit-diff exported.
- 2026-06-19 `128330e36` (#5899): fuzzy writes only touched lines.
- 2026-07-04 `a1b336d73` (#6278): extra replacement fields allowed.
- 2026-08-17 `ca21c1686` (commit #8011; CHANGELOG 0.84.3 cites #7835): single edit object → array.
- 2026-08-19 `1355cd36e` (#8337): BOM normalization in text inputs.
- 2026-09-02 `e583b290a` (#8979): write result "Successfully wrote N bytes" → "Successfully wrote to {path}" (count was UTF-16 code units) → [[tool-result-misreports-facts]].
- 2026-09-05 `fcff255b0`: strict-prefer constrained sampling default (was `PI_EXPERIMENTAL`-gated since `7915cdac6` 2026-08-11).
- 2026-09-01 `62835ea81` (#8627): `ctx.cwd`.
- 2026-09-29 `445770e03` / 2026-10-01 `7fd478a2e`: durable copy (Package 16) replaces `packages/agent/src/harness/tools/{edit,edit-diff,write,file-mutation-queue}.ts`.

## Evidence commits
`ffc9be886` `159075cad` `9e3e319f1` `52325adb9` `c214a3340` `8c43a9fbc` `1ed009e2c` `d9adf659c` `0c135d014` `74a46fc7e` `235b247f1` `20a57e759` `a0734bd16` `e773527b3` `b5f425ad1` `f7cd613ee` `a2ec01e12` `43ee9b77e` `ebdf3cf45` `60a55a239` `2b46f3886` `128330e36` `a1b336d73` `ca21c1686` `1355cd36e` `e583b290a` `7915cdac6` `fcff255b0` `62835ea81` `445770e03` `7fd478a2e`

## Quirks
- Not-found error still says "must match exactly including all whitespace and newlines" though a fuzzy fallback exists (`edit-diff.ts:253-262`).
- Uniqueness counted in fuzzy space: an `oldText` that is exactly unique can be rejected as "Found 2 occurrences" if another region differs only by trailing whitespace/quotes/dashes (`edit-diff.ts:247-251,328-331`).
- `prepareEditArguments` mutates the incoming args object in place (`edit.ts:108,115-122`); durable copy works on a copy.
- write/edit: if abort lands after `writeFile` resolves, the file is written but the model gets "Operation aborted" (`write.ts:81-82`, `edit.ts:198-199`) (observed from code).
- Error messages differ for single vs multi edits (index `edits[i]` only in multi).
- Durable `edit` and CA `edit-diff.ts` identical except BOM helper (per findings `diff`, unverified line-by-line).

## Durable variant (packages/durable)
- `packages/durable/src/tools/edit.ts`: same `edits[{oldText,newText}]` contract; `prepareArguments` repairs JSON-string / single object / legacy top-level on a **copy** (`:45-71`); checks path is file or symlink (`:106-110`); strip BOM, detect CRLF, LF-normalize, exact-then-fuzzy (`edit-diff.ts:203-244`), restore endings+BOM, write via `env.writeFile` (`edit.ts:116-125`); details `{diff, patch, firstChangedLine}` (`:73-77,127-137`).
- `write.ts:16-38`: inside `withFileMutationQueue`, abort checks before/after, `env.writeFile` creates parents.
- All I/O through `ExecutionEnv` ([[pluggable-tool-backends]]); queue keyed by `env.id + canonical path` ([[pi--per-file-mutation-queue|per-file-mutation-queue]]).
- No `replay` declared → default `unsafe`; crash mid-edit yields "interrupted, may have partially run" ([[crash-safe-tool-replay]]).

## Failures
- [[edit-tool-dual-mode-confusion]]
- [[tool-arg-shape-drift]]
- [[edit-invisible-character-mismatch]]
- [[concurrent-file-mutation-interleave]]
- [[tool-result-misreports-facts]]
