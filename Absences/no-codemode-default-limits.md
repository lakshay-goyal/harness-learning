---
type: absence
harnesses: [opencode]
---
# no-codemode-default-limits

Code mode ships without a timeout, tool-call cap or output cap. Limits are the host's job, and opencode's host passes none.

**What's missing**
- Library defaults: "absent means no timeout / unlimited / no truncation" (`packages/codemode/src/codemode.ts:11-16`).
- opencode's `execute` tool passes no `limits` (`packages/opencode/src/tool/code-mode.ts:239-260`). Only the generic 2000-line / 50 KB tool-output truncation applies, and the user's abort.
- Bounded pieces that do exist: `TOOL_CALL_CONCURRENCY = 8` (`packages/codemode/src/stdlib/promise.ts:6`), recursion guards `MAX_RENDER_DEPTH = 8`, `MAX_VALUE_DEPTH = 32` (`packages/codemode/src/tool-schema.ts:35`; `packages/codemode/src/tool-runtime.ts:122`).

**Evidence of decision** (`packages/codemode/codemode.md:130-143`, "Decisions and Rationale")
- "Leave execution-limit defaults to hosts. Appropriate budgets depend on the surrounding product and its own cancellation, retention, and output-bounding policies."
- "Keep an owned tree-walking interpreter. The product need is bounded tool orchestration, not arbitrary JavaScript." (rejected: QuickJS / `node:vm` / WASM).
- "Skip unsupported OpenAPI operations. Incorrect parameter encoding, authentication, or transport behavior is worse than a precise `skipped` reason."
- Package landed `2409c7a3d5` (2026-07-03, #35079), after a same-day revert of `cb93114424` (`379adee35c`).

**Contrast**
- pi's codemode (QuickJS) has 256 MiB heap, 300 s library timeout (but `Infinity` in coding-agent unless `timeout_ms`), 10k-token output budget, 4 concurrent model calls (`8562bcf66`; [[Constants]] Top 25 #22) → [[pi--code-mode|pi]].

**Implication**
- Owning the interpreter removes the "arbitrary JS" risk class, but not runaway loops. A script that loops over tool calls is bounded only by the user's Esc. Experimental flag only.

Related: [[code-mode]] · [[nested-tool-calls]] · [[mcp-integration]] · [[tool-output-truncation]] · [[opencode]] · [[Absences]]
