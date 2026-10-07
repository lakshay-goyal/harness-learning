---
type: home
---

Cross-harness atlas of coding-agent internals. Each harness strengthens shared concept notes. Schema: `.meta/schema.md`.

## Harnesses
| Harness | Commit | Studied | Digest |
|---|---|---|---|
| [[pi]] | `b30a6dd77` | 2026-10-07 | [[2026-10-07-pi]] |
| [[opencode]] | `ecc4916b5a` | 2026-10-08 | [[2026-10-08-opencode]] |

## Map
| Group | Concepts | Failures |
|---|---|---|
| 01 loop | [[Loop]] | [[Loop Failures]] |
| 02 model interface | [[Model Interface]] | [[Model Interface Failures]] |
| 03 tools | [[Tools]] | [[Tools Failures]] |
| 04 prompting | [[Prompting]] | [[Prompting Failures]] |
| 05 context | [[Context]] | [[Context Failures]] |
| 06 caching | [[Caching]] | [[Caching Failures]] |
| 07 safety | [[Safety]] | [[Safety Failures]] |
| 08 state | [[State]] | [[State Failures]] |
| 09 subagents | [[Subagents]] | [[Subagents Failures]] |
| 10 platform | [[Platform]] | [[Platform Failures]] |

Also: [[Constants]] · [[Absences]] · [[Tradeoffs]] (19 pi-vs-opencode axes) · `Digests/`

## Dashboards (Dataview plugin, optional)

Concepts by tier:
```dataview
TABLE tier, stage, length(harnesses) AS "#harnesses"
FROM "Concepts"
WHERE type = "concept"
SORT tier ASC, file.folder ASC
```

Must-haves (≥60% of studied harnesses, min 3 studied):
```dataview
LIST FROM "Concepts" WHERE type = "concept" AND tier = "must-have"
```

Failures per concept group:
```dataview
TABLE length(rows) AS failures
FROM "Failures"
WHERE type = "failure"
GROUP BY file.folder
```

Implementations by harness:
```dataview
TABLE concept, commit
FROM "Implementations"
WHERE type = "implementation"
SORT harness ASC, concept ASC
```

Absences:
```dataview
LIST FROM "Absences" WHERE type = "absence"
```

Recent digests:
```dataview
LIST FROM "Digests" SORT file.name DESC LIMIT 10
```
