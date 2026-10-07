---
type: absence
harnesses: [codex]
---
# no-collaboration-styles

Pair Programming / Execute / Custom collaboration styles were removed; only Default and Plan modes remain.

**What's missing**
- Templates in `codex-rs/collaboration-mode-templates/` cover Default and Plan only; feature key `collaboration_modes` is `Stage::Removed`.

**Evidence of decision**
- `31415ebfcf` 2026-01-19; `d509df676b` 2026-02-03.

**Implication**
- Mode proliferation confused models across modes ([[mode-state-confusion]]); the surviving split is read-only planning vs doing ([[plan-mode]]).

Related: [[plan-mode]] · [[mode-state-confusion]] · [[no-plan-mode]] · [[Absences]]
