---
type: failure
concepts: [secret-handling, layered-settings]
harnesses: [opencode]
---
**Symptom** — Users who kept API keys out of `opencode.json` with `{env:API_KEY}` found the real key written into the file: when the config lacked `$schema`, the loader added it by re-serializing the parsed config, whose `{env:…}` references had already been substituted — secrets landed in a file often committed to git.

**Root cause** — The file was round-tripped through its interpolated in-memory form.

**Fix · [[opencode]]** — `052f887a9a` 2026-01-17 "prevent env variables in config from being replaced with actual values": `$schema` is inserted by textual replace on the original text (`packages/opencode/src/config/config.ts:245-247`).

**Lesson** — Never write a config file back from its interpolated form; edit the original text or keep references unresolved.

Related: [[secret-handling]] · [[layered-settings]] · [[config-value-indirection-ambiguity]] · [[opencode--secret-handling|opencode]]
