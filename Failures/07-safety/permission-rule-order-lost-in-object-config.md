---
type: failure
concepts: [permission-ruleset, layered-settings]
harnesses: [opencode]
---
**Symptom** — Users wrote permission config such as `{"bash": {"*": "ask", "git status": "allow"}}` expecting the later, more specific rule to win (last-match-wins). After config decoding the effective order differed from the written order, so allow/deny precedence silently changed.

**Root cause** — The ruleset semantics depend on order, but the config representation was a JSON object; key order did not survive schema decode and merging, and layered config files were merged as maps.

**Fix · [[opencode]]** — `66f93035b0` 2026-04-25 "fix permission config order" (#24222); `a9740b9133` 2026-04-25 "preserve permission order with Effect decode" (#24308); `65368f609d` 2026-05-12 "preserve permission ordering by accepting a layered array" (#23214): permission config may be an ordered array of layers (`65368f609d:packages/opencode/src/config/permission.ts`; schema now at `packages/core/src/v1/config/permission.ts`). v2 config authors permissions only as ordered arrays (`specs/v2/config.md:294-303`).

**Lesson** — An order-sensitive rule language needs an ordered representation end to end: file, decoder, merge and evaluation.

Related: [[permission-ruleset]] · [[layered-settings]] · [[opencode--permission-ruleset|opencode]]
