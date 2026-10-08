---
type: failure
concepts: [layered-settings, spec-driven-agentic-development]
harnesses: [opencode]
---
**Symptom** — (Latent, observed at `ecc4916b5a`; minor.) The in-tree agent guide for the LLM adapter says "`OPENCODE_EXPERIMENTAL_NATIVE_LLM=true` or the umbrella `OPENCODE_EXPERIMENTAL=true` opts in" (`packages/opencode/src/session/llm/AGENTS.md:88`), but setting only the umbrella flag does not enable the native runtime.

**Root cause** — `dbe36851bc` 2026-05-18 (#27114) wrote the guide while the flag followed the umbrella. `db63eaf6ea` 2026-05-24 (#29123) "make OPENCODE_EXPERIMENTAL_NATIVE_LLM separate from OPENCODE_EXPERIMENTAL" switched it to a plain `bool(...)` (`packages/opencode/src/effect/runtime-flags.ts:54`) instead of `enabledByExperimental(...)` (`packages/opencode/src/effect/runtime-flags.ts:11-14`), and updated the user docs (`packages/web/src/content/docs/cli.mdx` and translations) but not the agent-facing `AGENTS.md` next to the code.

**Fix · [[opencode]]** — none at HEAD.

**Lesson** — Directory-scoped agent guides are documentation that agents act on; a flag change needs the same grep-and-update pass over `AGENTS.md` files as over user docs, ideally with a check that every flag named in docs maps to its real resolver.

Related: [[layered-settings]] · [[spec-driven-agentic-development]] · [[telemetry-docs-outlive-code]] · [[opencode]]
