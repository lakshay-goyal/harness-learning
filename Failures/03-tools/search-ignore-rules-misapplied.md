---
type: failure
concepts: [search-tools]
harnesses: [pi]
---
**Symptom** — `find` hid files it should show: rules from `a/.gitignore` hid files under sibling `b/`; a parent repo's `.gitignore` hid files inside a nested git repo.

**Root cause** — pi collected every nested `.gitignore` and passed them via `--ignore-file`, which fd applies **globally**; later `--no-require-git` was passed even inside repos, so parent ignore rules crossed nested-repo boundaries.

**Fix · [[pi]]**
- `1d4fdbad2` 2026-04-16 (#3303) — stop manual collection; delegate hierarchical ignore semantics to fd.
- `756a4e8f6` 2026-06-22 (#5960) — `--no-require-git` only when **not** inside a git repo (walk up for `.git`) (`packages/coding-agent/src/core/tools/find.ts:184-198`).

**Lesson** — Ignore files are hierarchical and repo-scoped; use the backend's native hierarchy instead of flattening rules.

Related: [[search-tools]] · [[find-glob-semantics-mismatch]] · [[pi--search-tools|pi]]
