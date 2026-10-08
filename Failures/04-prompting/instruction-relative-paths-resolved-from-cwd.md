---
type: failure
concepts: [skill-progressive-disclosure, self-documentation-pointer]
harnesses: [pi]
---
**Symptom** — Relative paths inside harness-supplied instructions were resolved against the user's project cwd: SKILL.md references (e.g. a hypothetical `scripts/foo.sh`, illustrative only) looked up in the project (#1136, #1171); pi doc cross-references (`docs/extensions.md`, `examples/...`) looked up in the user's repo (#4752).

**Root cause** — Models default to cwd for any relative path; the prompt gave the location of the SKILL.md / docs root but no resolution rule.

**Fix · [[pi]]**
- Skills: `09bca9672` had "{baseDir} placeholders" (dropped `05b7b8133`); `3c687b427` 2026-02-01 (#1136) path-resolution guidance in skills preamble; `5d6a7d6c3` 2026-02-02 (#1171) better skill dir resolution. HEAD: "When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands." (`packages/coding-agent/src/core/skills.ts:372`); `/skill:` invocation adds "References are relative to <baseDir>." (`packages/coding-agent/src/core/agent-session.ts:2159`).
- Docs: `48b6510c1` 2026-05-19 (#4752): "When reading pi docs or examples, resolve docs/... under Additional docs and examples/... under Examples, not the current working directory" (`system-prompt.ts:166`); paths are absolute (`config.ts:444-456`).

**Lesson** — State the base directory for every relative reference you hand the model; models resolve against cwd by default.

Related: [[skill-progressive-disclosure]] · [[self-documentation-pointer]] · [[path-normalization]] · [[pi--skill-progressive-disclosure|pi]]
