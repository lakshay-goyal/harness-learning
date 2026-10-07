---
type: concept
stage: tools
tier: candidate
aliases: [Format.Service, format.file, formatter config, "formatter: false"]
harnesses: [opencode]
---
After a mutating tool writes a file, the harness runs a formatter and re-reads the file, so the diff and the model's view match the formatted bytes.

## Why
- Models rarely match project formatting exactly; unformatted edits create noisy diffs and failing lint.
- If the harness formats but does not re-read, the model's next `oldString` targets bytes that no longer exist.
- Formatting rewrites user files outside the edited span, so it must be controllable.

## Design space
- **Built-in formatter table detected per file extension/project marker** (opencode, ~26 formatters) vs user-configured commands only.
- Re-read after format so the returned diff is post-format (opencode `edit`).
- Default: on (opencode until 2026-04-16) vs **opt-in by config** (opencode since `220e3e9a2b`).
- Leave formatting to the model via shell (pi; no formatter hook).

## Implementations
- [[opencode--post-edit-formatting|opencode]] — `format.file(path)` after edit/write/apply_patch; edit re-syncs content with BOM handling; disabled unless `formatter` is set.

## Failures
- (none recorded)

## Tradeoffs
- [[lsp-feedback-vs-none]]

## Related
[[lsp-diagnostics-feedback]] · [[search-replace-edit]] · [[patch-envelope-edit]] · [[fuzzy-edit-matching]]
