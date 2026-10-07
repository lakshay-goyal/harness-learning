---
type: implementation
harness: opencode
concept: spec-driven-agentic-development
commit: ecc4916b5a
files: [CONTEXT.md:1-225, specs/v2/instructions.md:7-14, specs/v2/instructions.md:77, specs/v2/todo.md:3, specs/v2/schema-changelog.md:39-41, AGENTS.md:157, packages/core/src/session/runner/llm.ts:46-47, specs/project.md:3]
---
[[spec-driven-agentic-development]] in [[opencode]].

## Mechanism
### v2 runtime (the spec-driven rebuild)
- **Domain glossary** `CONTEXT.md` (225 lines): each term defined once with `_Avoid_:` synonyms (9 of them, e.g. "_Avoid_: System prompt") so agents and humans use one vocabulary; also holds client/stream contracts (`CONTEXT.md:156-170`).
- **Specs** `specs/v2/*.md` (instructions, session, tools, config, provider-model, provider-policy, catalog-config-plugin-lifecycle, todo, schema-changelog, `api.html`): 39 commits touch `specs/v2`, 14 touch `CONTEXT.md` (`git log --oneline`).
- **Standing direction for agents**: "Move behavior out of large application services and into plugins…" (`specs/v2/instructions.md:7`); "Services are hot-reloadable by design" (`specs/v2/instructions.md:13`); "`packages/opencode` becomes thinner over time" (`specs/v2/instructions.md:14`); "Avoid moving legacy services over wholesale" (`specs/v2/instructions.md:77`).
- **Guard rails in code comments and root rules**: runner header "Keep this as orchestration over smaller collaborators rather than rebuilding the legacy `SessionPrompt` monolith. Implement the unchecked items in small reviewed slices" (`packages/core/src/session/runner/llm.ts:46-47`); root `AGENTS.md:157` "Do not bridge through legacy `SessionPrompt.loop(...)`".
- **Contract log**: `specs/v2/schema-changelog.md` records every database, durable-event, projected-message, HTTP and SDK schema change with reason and compatibility (`specs/v2/schema-changelog.md:39`); branch of origin `feat/opencode-embedded-api` (`specs/v2/schema-changelog.md:41`). Pre-launch event stores are "disposable rather than compatibility targets" (`specs/v2/session.md:173`).
- **Goal statement**: `specs/v2/todo.md:3` "ok we need to work towards a launch of v2 so we can get out of this rebuild phase". Multi-project single process goal dates to `specs/project.md:3` (2025-09).

### Legacy runtime
- Still owns the shipping loop; imports core v1 schemas (`83452558f7` 2026-06-02) and shares storage, prompts and provider code with v2 at a migration seam ([[client-server-session-split]]).

## Constants
| name | value | path:line |
|---|---|---|
| `_Avoid_:` glossary entries | 9 | `CONTEXT.md` |

## Evolution
- 2025-09-01 `f993541e0b` multiple instances in one process (spec `specs/project.md`).
- 2026-05-08 `5bb7b23440` native LLM core foundation (`packages/llm`).
- 2026-06-03 `76ee87ead8` embedded v2 session runtime + `CONTEXT.md` v2 glossary + `specs/v2/session.md` + `schema-changelog.md`.
- 2026-06-06 `660a00d317` unified v2 tool architecture (`specs/v2/tools.md`).
- 2026-06-26 `65210f2d97` durable session history pages; 2026-06-30 `a4b6047e64` drop legacy config filename (last `specs/v2/config.md` change).
- Design churn concentrates 2026-06-03 → 2026-06-26; afterwards code-only (runner last touched `e9f8a210b9` 2026-09-30).

## Quirks / drift
- Specs barely changed after 2026-06-30 while v2 code kept moving to 2026-09-30, so drift is likely (inferred from commit dates; not audited).
- Governance: "any UI or core product feature must go through a design review with the core team before implementation" (`CONTRIBUTING.md:13`).

pi contrast: pi writes one normative spec + ordered package handoff per subsystem (Pico5, pi-durable) rather than a standing glossary + changelog ([[pi--spec-driven-agentic-development|pi]]).
