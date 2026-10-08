---
type: failure
concepts: [search-tools, path-normalization]
harnesses: [pi]
---
**Symptom** — `find` with a path glob like `src/**/*.spec.ts` silently returned **nothing**; path globs returned nothing on Windows; searching from a filesystem root (`/`, `I:\`) dropped the first character of every result path.

**Root cause** — fd `--glob` matches the basename only unless `--full-path`; Windows separators; results relativized by slicing at `searchPath.length+1`, which is wrong when the root already ends with a separator.

**Fix · [[pi]]**
- `c5451af74` 2026-04-16 (#3302) — patterns containing `/` → `--full-path` and prepend `**/` unless already anchored (`packages/coding-agent/src/core/tools/find.ts:201-209`).
- `d4eaf052b` 2026-08-05 (#6817) — Windows: `/` → `[/\\]` in full-path patterns (`find.ts:211-212`).
- `523b5a491` 2026-08-04 (#7569; findings cite #6104) — `path.relative`, trailing separator preserved (`find.ts:14-24`).

**Lesson** — Models write globs the way shells/IDEs accept them; adapt the backend invocation to that, and never relativize paths by string slicing.

Related: [[search-tools]] · [[path-normalization]] · [[search-ignore-rules-misapplied]] · [[pi--search-tools|pi]]
