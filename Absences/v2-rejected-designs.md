---
type: absence
harnesses: [opencode]
---
# v2-rejected-designs

What opencode's v2 runtime (`packages/core`, `packages/llm`, specified in `specs/v2/**`) deliberately does **not** carry over or does not build yet. Each line quotes the spec.

**Not ported (rejected)**
| design | v2 stance | evidence |
|---|---|---|
| Custom commands | "V2 does not expose separate user-authored command configuration. Skills should cover named reusable prompt workflows"; "intentionally does not port legacy command-only behavior such as per-command `model`, `agent`, `subtask`, prompt shell expansion, or positional/template substitution" | `specs/v2/config.md:48-50` → [[prompt-template-expansion]], [[skill-progressive-disclosure]] |
| Legacy `config.json` filename | "not supported in V2" | `specs/v2/config.md:16` |
| Singular `provider` key | plural `providers`, "v2 does not add a compatibility alias" | `specs/v2/config.md:206` |
| Implicit hidden-file disclosure | "Hidden-file discovery is intentionally narrower than an unconditional ripgrep `--hidden` traversal" (legacy grep passes `--hidden`) | `specs/v2/schema-changelog.md:225,230` → [[search-tools]] |
| Tools truncating their own output | "Tools return complete validated domain output. They do not truncate model-facing output or manage retention files"; one settlement boundary bounds the result | `specs/v2/tools.md:155` → [[tool-output-truncation]] |
| Registry-level authorization | "The registry does not inject an `assertPermission` helper"; catalog filtering is visibility, not authorization | `specs/v2/tools.md:131` → [[permission-ruleset]] |
| Background bash | "The model has no registered observation or cancellation tool for background bash jobs" | `specs/v2/schema-changelog.md:697` → [[no-background-bash]] |
| Pretend sandbox | mutation checks "without pretending path APIs provide a syscall-level sandbox" | `specs/v2/schema-changelog.md:270` → [[no-sandbox]] |
| apply_patch moves + atomic rollback | "Sequential semantics are small and honest: they avoid claiming rollback or transactionality that path-based filesystem commits do not provide"; "Moves and atomic rollback are deliberately unsupported in the first slice" | `specs/v2/schema-changelog.md:613,618` → [[patch-envelope-edit]] |

**Deferred (not yet)**
| design | v2 stance | evidence |
|---|---|---|
| Fuzzy edit | "Richer V1 fuzzy edit behavior remains intentionally deferred"; exact match only | `specs/v2/schema-changelog.md:275`; `packages/core/src/tool/edit.ts:84` → [[fuzzy-edit-matching]] |
| Provider timeout / watchdog | "Provider timeout, retry, and watchdog policy is intentionally deferred. The runner does not impose a universal provider-stream inactivity or absolute timeout." | `specs/v2/session.md:153` → [[http-transport-hardening]] |
| Post-crash auto-continuation | "Post-crash continuation recovery is intentionally deferred" | `specs/v2/session.md:165` → [[durable-execution]] |
| Per-turn tool-call limits | "Eager local-tool execution is intentionally unbounded in the current local slice" | `specs/v2/session.md:173` |
| Session retry + doom-loop guard | runner TODO "[ ] Bound provider retries and repeated identical tool calls" | `packages/core/src/session/runner/llm.ts:55` → [[repeated-tool-call-detection]], [[auto-retry-backoff]] |
| Manual compaction, tool-output pruning | `compact` returns `OperationUnavailableError`; "Deterministic old tool-result pruning remains a separate follow-up" | `packages/core/src/session.ts:419`; `specs/v2/session.md:121` → [[tool-output-pruning]] |

**Implication**
- v2 trades feature parity for honest contracts: no claim of rollback, sandboxing or authorization that the mechanism cannot back. The cost is regressions against legacy (no doom-loop guard, no session retry, no fuzzy edit) while both runtimes ship.
- Several "deferred" items are exactly the guards legacy learned from failures ([[identical-tool-call-loop]], retry caps from `c78986831c`).

Related: [[event-sourced-session-store]] · [[transcript-carried-system-prompt]] · [[spec-driven-agentic-development]] · [[opencode]] · [[Absences]]
