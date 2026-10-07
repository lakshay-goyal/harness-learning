---
type: failure
concepts: [permission-ruleset, skill-progressive-disclosure]
harnesses: [opencode]
---
**Symptom** — An agent whose permissions hide some skills could still learn their names: asking for a non-existent skill returned an error listing every skill in the instance / Location, including the denied ones.

**Root cause** — The listing shown to the model was permission-filtered, but the not-found error path built its "Available skills: …" text from the unfiltered catalog. Two code paths, one filter.

**Fix · [[opencode]]**
- v2 `3f64b5e621` 2026-06-05 (#30843) "admit v2 skill guidance": skill list moved out of the Location-wide `skill` tool description into permission-filtered per-agent guidance, and the not-found error became `Unable to load skill <name>` with no catalog (`packages/core/src/tool/skill.ts:54-55,74`); changelog entry "Stop missing-skill errors from enumerating the unfiltered Location-wide skill catalog." (`specs/v2/schema-changelog.md:813`).
- Still live in legacy at HEAD: `Skill.require` throws `NotFoundError({ name, available: Object.keys(s.skills).toSorted() })` (`packages/opencode/src/skill/index.ts:294-299`), whose message is `Skill "<name>" not found. Available skills: …` (`packages/opencode/src/skill/index.ts:73-80`); the `skill` tool re-throws that message (`packages/opencode/src/tool/skill.ts:23-25`). The model-facing listing uses `available(agent)`, which drops `deny` skills (`packages/opencode/src/skill/index.ts:310-315`).

**Lesson** — Error text is model input too: anything an error enumerates must pass the same permission filter as the listing it shadows.

Related: [[permission-ruleset]] · [[skill-progressive-disclosure]] · [[opencode--permission-ruleset|opencode permissions]] · [[internal-errors-leak-over-wire]]
