---
type: implementation
harness: codex
concept: overflow-recovery
commit: 622e9e3696
files: [codex-rs/core/src/session/turn.rs:1688, codex-rs/core/src/session/turn.rs:816, codex-rs/core/src/session/turn.rs:758, codex-rs/core/src/session/turn.rs:1318, codex-rs/core/src/compact.rs:330, codex-rs/core/src/context_manager/history.rs:494, codex-rs/core/src/context_manager/history.rs:680]
---
[[overflow-recovery]] in [[codex]] — recovery is deferred to the next turn, not done in place.

## Mechanism
- **Normal sampling overflow**: `ContextWindowExceeded` → `set_total_tokens_full` then return the error (`codex-rs/core/src/session/turn.rs:1688-1691`); it falls into the generic `Err(e)` arm that emits an error and ends the turn ("let the user continue the conversation", `:816-831`). Because usage is pinned to the full window (`set_token_usage_full(context_window)`, `codex-rs/core/src/context_manager/history.rs:494-501`), the next turn's pre-sampling check (`codex-rs/core/src/session/turn.rs:1318`) compacts first. No generic compact-and-retry ([[no-in-turn-overflow-retry]]).
- **Approval-review (guardian) sessions**: the only overflow → compact → retry path, once per model step (`guardian_budget_compacted`, `ExhaustedReviewBudget`, `codex-rs/core/src/session/turn.rs:758-787`); token-budget sessions must fail closed instead ("token-budget resets do not summarize", `:336-342`).
- **Compaction request overflow** (local): `history.remove_first_item()` and retry with the retry counter reset — "Trim from the beginning to preserve cache (prefix-based) and keep recent messages intact" (`codex-rs/core/src/compact.rs:330-346`); the counterpart call/output of the removed item is removed too (`codex-rs/core/src/context_manager/history.rs:680-692`); one item left → `set_total_tokens_full` and fail (`codex-rs/core/src/compact.rs:340`). Remote compaction instead pre-trims tool outputs newest-first ([[codex--auto-compaction]]).
- **Invalid image** from the provider: turn ends with "Invalid image in your last message. Please remove it and try again." — no automatic removal (`codex-rs/core/src/session/turn.rs:793-815`) → [[codex--image-normalization]].
- **Prevention**: 90 % auto-compact limit, 95 % effective window, opt-in post-turn compaction, compaction before switching to a smaller-window model ([[codex--auto-compaction]]). Overflow detection itself (error codes) → [[context-overflow-detection]].

## Evolution
- 2025-10-08 `687a13bbe5` (#4942) "truncate on compact … iteratively" — drop oldest item and retry on ContextWindowExceeded → [[summarization-request-overflows]].
- 2025-10-20 `049a61bcfc` auto-compact at ~90 % ("Users now hit a window exceeded limit and they usually don't know what to do").
- 2026-05-11 `15e79f3c26` "[codex] Harden overflow auto-compaction recovery (#22141)": dogfooders hit a request that "could keep re-entering auto-compaction indefinitely if the compacted retry still did not fit" and a local rebuild that flattened `[image, "what is this?"]` so the retry lost the image; moved overflow handling into the sampling retry loop. Reverted the same day `69f3183a8e` (#22170) → [[overflow-compaction-cascade]].
- 2026-07-07 `172ab264bd` previous-model compaction rejected → retry with current model → [[compaction-pinned-to-unavailable-model]].

## Versus pi
- [[pi--overflow-recovery]]: hide the failed attempt, compact, retry exactly once (latch). codex: fail the turn, pin usage, compact before the next turn; relies on proactive thresholds instead of reactive recovery.
