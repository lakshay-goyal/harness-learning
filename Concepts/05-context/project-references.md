---
type: concept
stage: context
tier: variant
aliases: ["<available_references>", Reference.Service, ReferenceGuidance, "core/reference-guidance", "references config", project reference]
harnesses: [opencode]
---
Extra named directories outside the workspace advertised to the model (name/path/description) so it can read them on demand.

## Why
- Real tasks need sibling repos, vendored SDKs or docs checkouts; without an advertised list the model guesses paths or asks, and file tools may refuse paths outside the project.
- Listing only name/path/description keeps the prompt small: contents are read lazily with ordinary tools ([[skill-progressive-disclosure]] for directories).
- Advertising a path is useless unless permissions also allow reading it (opencode auto-allows reference dirs under `external_directory`).

## Design space
- **Source**: local path · remote git repo materialized into a harness-owned cache and refreshed in the background (opencode `Reference` git source → `Repository.cachePath(global.repos, …)`).
- **Visibility**: listed only when it has a description (opencode) · `hidden` flag (opencode).
- **Placement**: block in the system prompt (opencode legacy, rebuilt each step) · typed [[transcript-carried-system-prompt]] source with update/removal renderers so changes append instead of rewriting the prefix (opencode v2 `core/reference-guidance`).
- **Permission coupling**: reference dirs whitelisted for out-of-workspace reads (opencode) · separate prompt ([[workspace-boundary-check]]).
- **Alternative tried**: a research subagent with repo clone/overview tools (opencode `scout` agent, added `40d5ea1cf1` 2026-05-09, removed `a639fe7a08` 2026-06-02; reason not stated — unverified).

## Implementations
- [[opencode--project-references|opencode]] — `references` config → `Reference.Service` (local or git-cached); `<available_references>` block sorted by name; dirs auto-allowed for `external_directory`; v2 as a Context Source.

## Failures
- (none recorded)

## Related
[[workspace-boundary-check]] · [[context-file-hierarchy]] · [[skill-progressive-disclosure]] · [[transcript-carried-system-prompt]] · [[cache-stable-prompt-prefix]] · [[env-vars-as-context]]
