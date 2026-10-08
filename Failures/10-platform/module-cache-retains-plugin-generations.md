---
type: failure
concepts: [runtime-plugin-loading]
harnesses: [pi]
---
**Symptom** — Each hot reload of experimental Chord facet plugins leaked the previous generation's code and closures; memory grew with every `/reload`.

**Root cause** — Node's ESM loader caches every imported module URL forever; content-addressed bundles produce new URLs, so old generations can never be collected ("retains every imported module generation", `packages/chord/PLANNING.md:187`).

**Fix · [[pi]]** — `65f77ecd6` 2026-08-31: facet bundles emitted as CommonJS and evaluated with `node:vm` `compileFunction`, outside the ESM cache (`packages/chord/src/node/bundle-loader.ts:7,192`), after integrity check (`:25,233-240`).

**Lesson** — Hot reload in Node needs a loader outside the module cache; verify generations are actually garbage-collectable.

Related: [[runtime-plugin-loading]] · [[reload-cannot-drain-in-flight-calls]] · [[pi--runtime-plugin-loading|pi]]
