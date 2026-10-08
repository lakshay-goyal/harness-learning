# Vault schema

```
harness-atlas/
  Home.md            dashboards (Dataview) + static index
  Constants.md       cross-harness constants table
  Concepts/<NN-group>/  one note per harness-neutral concept, grouped:
      01-loop 02-model-interface 03-tools 04-prompting 05-context
      06-caching 07-safety 08-state 09-subagents 10-platform
      each group folder has an overview note (e.g. `Loop.md`, type: group)
  Harnesses/         one note per harness
  Implementations/<harness>/<harness>--<concept>.md
  Failures/<NN-group>/  failures, grouped by the group of their first concept;
      each has `<Group> Failures.md` overview (type: group) linked from the concept group note
  Tradeoffs/         design axes where harnesses diverge
  Absences/          deliberate non-features / removed designs + rationale
  Digests/           dated run digests
  _meta/schema.md    copy of this file
  _templates/
```

File names: kebab-case. Concept = `<concept>.md`. Implementation = `<harness>--<concept>.md` (unique basenames so `[[links]]` never collide).
New concept → put in the best-fitting group folder and add a line `- [[slug]] — <definition>` to that group's overview note. Create a new group only if ≥ 3 concepts fit none (next number, add to Home map).
Links: concept → `[[<harness>--<concept>|<harness>]]`; implementation → `[[<concept>]]` + `[[<harness>]]`.
Merging a harness into an existing concept: add to `harnesses:`, append its link to **Implementations**, extend **Design space** only if it adds an option.

## Color convention

Rendering only, driven by link targets. Notes stay plain markdown — no inline HTML,
no color spans, ever. Write `[[wikilinks]]` per the rules above and colors follow.

| Kind | Inside notes | Graph node |
|---|---|---|
| Harness topic `[[pi]]` | red | red (`path:"Harnesses"` group) |
| Harness implementation `[[pi--turn-loop\|pi]]` | red | default |
| Concept / group overview `[[turn-loop]]`, `[[Loop]]` | blue | blue (`path:"Concepts"` group) |
| Failure, tradeoff, absence, digest | default | default |

Mechanisms are separate and neither covers the other:
- notes → `.obsidian/snippets/harness-colors.css`, regenerate with
  `python3 .meta/generate-link-colors.py`
- graph → `colorGroups` in `.obsidian/graph.json` (CSS cannot target SVG graph nodes)

Re-run the generator after any note is added or renamed.

## Concept
```yaml
---
type: concept
stage: loop | model-interface | messages | tools | tool-design | context | caching | compaction | permissions | state | architecture | subagents | cost | failure-handling | eval
tier: candidate | must-have | variant
aliases: [harness-specific terms]
harnesses: [opencode]
---
```
Body: one-line definition · **Why** (what breaks without it) · **Design space** (options) · **Implementations** (links) · **Related**.

## Implementation
```yaml
---
type: implementation
harness: opencode
concept: prefix-stability
commit: 18ef3cc7c5
files: [packages/.../file.ts:123]
---
```
Body: mechanism in bullets · constants · evidence commits · quirks. Link `[[concept]]` and `[[harness]]`.

## Failure
```yaml
---
type: failure
concepts: [tool-description-design]
harnesses: [opencode]
---
```
Body: **Symptom** (what the model did) · **Fix** per harness with hash · **Lesson** (one line, generalized).

## Tradeoff
```yaml
---
type: tradeoff
concepts: [...]
---
```
Body: axis · table of options × harnesses · when each wins.

## Harness
```yaml
---
type: harness
repo: https://...
commit: <sha>
language: TypeScript
studied: YYYY-MM-DD
---
```
Body: ideology in 3 bullets · organ map table (organ → files) · distinctive choices · absences · link to digest.

## Absence
```yaml
---
type: absence
harnesses: [opencode]
---
```
Body: what's missing · evidence of the decision · implication.

## Failure merge rule
Same model behavior in another harness → append a `**Fix · [[<harness>]]**` line and add the harness to `harnesses:`. Never a second note.

## Constants.md
One row per constant, one column per harness. New harness = new column; `—` where absent.
