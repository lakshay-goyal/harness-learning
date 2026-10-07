---
type: concept
stage: messages
tier: candidate
aliases: ["<docs>", "Pi documentation", getDocsPath, getReadmePath, customize-opencode]
harnesses: [pi, opencode]
---
The prompt carries absolute paths to the harness's own docs/examples plus a topic→file map, read on demand only when the user asks about the harness itself.

## Why
- Users ask the agent to extend/configure the harness ("add a custom provider", "write an extension"); without docs the model hallucinates APIs.
- Inlining docs is too expensive; a pointer costs a few hundred tokens and is cache-amortized.
- Relative doc paths get resolved against the user's project ([[instruction-relative-paths-resolved-from-cwd]]); truncated reads make the model act on partial docs ([[partial-file-read-acted-on]]).

## Design space
- No self-docs (model guesses).
- Single README pointer (pi Nov 2025).
- **Absolute paths + topic routing map + read-completely/follow-links rules + scope gate "only when the user asks about pi"** (**pi chose**).
- Pseudo-URLs (`pi-internal://README.md`) for provider-allowlisted static prompts (pi, 1 day in Jan 2026).
- Offload tool reference into a doc (codemode → `docs/codemode.md`).
- Measure the lift with paired with/without-docs evals (pi `packages/evals`).
- Ship harness docs as a built-in skill scoped to config edits (opencode).
- Docs URL fetched at runtime via the web tool (opencode legacy).

## Implementations
- [[pi--self-documentation-pointer|pi]] — `<docs>` section, 13-topic map, ~45% of default prompt.
- [[opencode--self-documentation-pointer|opencode]] — docs URL + WebFetch instruction in anthropic.txt and meta.txt; built-in `customize-opencode` skill with real config schemas (origin/v2 drops the URL line).

## Failures
- [[instruction-relative-paths-resolved-from-cwd]]
- [[partial-file-read-acted-on]]

## Related
[[minimal-system-prompt]] · [[tool-description-design]] · [[harness-evals]] · [[skill-progressive-disclosure]] · [[file-read-tool]]
