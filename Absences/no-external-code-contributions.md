---
type: absence
harnesses: [codex]
---
# no-external-code-contributions

The project does not accept external pull requests.

**What's missing**
- `docs/contributing.md:5`: "**We do not accept external code contributions or pull requests.**" with rationale (architectural context, roadmap) at `docs/contributing.md:7-11`.

**Evidence of decision**
- Policy text `31f23b6022` 2026-08-17; guidelines update `6a279f6d77` 2026-01-26; stale-PR auto-close workflows `6cda3de3a4` / `c95bd345ea` 2025-11-13. Public repo is a mirror of an internal monorepo: every commit since 2026-07-10 carries a `GitOrigin-RevId:` trailer.

**Implication**
- 663 distinct authors are mostly staff + bots; many commits co-authored by Codex itself (`Co-authored-by: Codex <noreply@openai.com>`, e.g. `7a8407bbb6`). Contrast pi's "PRs that bloat the core will likely be rejected" (`CONTRIBUTING.md:7-9`).

Related: [[replaceable-builtin-extension]] · [[Absences]]
