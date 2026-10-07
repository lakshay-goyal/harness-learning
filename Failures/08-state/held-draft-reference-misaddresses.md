---
type: failure
concepts: [replicated-state]
harnesses: [pi]
---
**Symptom** — In Chord's delta tracker, after an `unshift`, a write through a previously held element reference edited its *neighbour*; a write after a `splice` could emit a path the replica cannot resolve — replicas silently diverged from the producer.

**Root cause** — Draft proxies baked the array index/path at creation time; structural mutations (unshift/splice) shifted positions under held references. Value-diffing baselines were immune; path-recording proxies were not.

**Fix · [[pi]]** — `c4289b20e` 2026-09-18 "keep tracked paths correct across structural mutation": cells with a parent link walked at use time; one shared wrapper per target with a cell per position. Later the tracker became an immutable overlay with prepare/adopt (`9a139c62b` 2026-09-23; `packages/chord/src/delta/README.md:124-136`).

**Lesson** — Path-baked proxies are unsound under structural mutation; resolve paths at write time or diff values.

Related: [[replicated-state]] · [[settled-draft-then-probe-throws]] · [[pi--replicated-state|pi]]
