---
type: absence
harnesses: [codex]
---
# no-client-price-table

No per-model token prices in the client; USD cost comes from a server turn-cost endpoint.

**What's missing**
- Turn cost fetched by `codex-rs/app-server/src/turn_cost_worker.rs` via `codex-rs/backend-client/src/client/chatgpt_turn_cost.rs`.

**Evidence of decision**
- Observed design (M2); no rationale commit found.

**Implication**
- Cost visible only for providers exposing that endpoint, and only after settlement; contrast pi's catalog prices ([[usage-cost-accounting]]). Budgets are token-weighted instead ([[session-token-budget]]).

Related: [[usage-cost-accounting]] · [[session-token-budget]] · [[subscription-usage-limits]] · [[Absences]]
