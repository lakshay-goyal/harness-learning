---
type: implementation
harness: pi
concept: harness-evals
commit: b30a6dd77
files: [packages/evals/README.md:7-129, packages/evals/vitest.evals.config.ts:7-31, packages/evals/src/harness.ts:43-528, packages/evals/src/plan.ts:4-56, packages/evals/src/report.ts:9-483, packages/evals/src/cli.ts:24-192, packages/evals/src/docker.ts:39-175, packages/evals/docker/Dockerfile:1-44, packages/evals/docker/entrypoint.ts:27-143, packages/durable/src/testing/storage-conformance.ts:94-1661, packages/durable/src/testing/env-conformance.ts:100-466, packages/durable/src/testing/storage-benchmark.ts:24-455]
---
[[harness-evals]] in [[pi]].

## Mechanism
### `packages/evals` (`@earendil-works/pi-evals` v1.0.4, private)
- Built on vitest-evals 0.15.0 + `@vitest-evals/core`, autoevals 0.3.0, vitest 4.1.11 (`packages/evals/package.json:17-21`).
- Two kinds by suffix (`packages/evals/README.md:7-10`): `*.docs.eval.ts` = **documentation-lift** (each case in isolated `without_docs` and `with_docs` Docker containers; report gives lift); other `*.eval.ts` = **host** evals (plain vitest-evals on host). Projects `docs`/`host`: `pool:"forks"`, `maxWorkers:1`, no file parallelism, `testTimeout`/`hookTimeout` 300 000 ms (`vitest.evals.config.ts:7-31`).
- **No public benchmark** (no SWE-bench/terminal-bench anywhere); pi measures (a) whether its own docs help the model customize pi, (b) whether docs match implementation.
- Format: one `describeEval(set, {harness, judges, judgeThreshold:null}, it => it(case, ({run}) => run(input)))` per file (`packages/evals/README.md:111-127`); case id `"<set> > <case>"`, exactly two parts, duplicates rejected (`src/plan.ts:21-37`); input `string | Array<{type:"prompt"}|{type:"reload"}>` — `reload` calls `session.reload()` so agent-written extensions/config load (`harness.ts:43,374-380`); options `model`, `noTools`, `tools`, `customTools`, `workspaceFiles` (path-escape checked), `transformSystemPrompt`, `expectedPiDocumentation`, typed `output(...)` projection (`harness.ts:50-68,225-236`); `EvalTask = {file, fullName, evalSet, caseId, variant, model, runNumber}` (`plan.ts:4-15`).
- Eval files at HEAD:
  | file | kind | scenario | grader |
  |---|---|---|---|
  | `evals/smoke.eval.ts` | host | "capital of France", `noTools:"all"` | `expect` "Paris" + usage provider/model match + tokens>0 |
  | `evals/documentation-audit.eval.ts` | host | one case per `packages/coding-agent/docs/**/*.md`; model audits page vs implementation with read/grep/find/ls + terminating `submit_documentation_audit` (`constrainedSampling json_schema strict:"prefer"`, `terminate:true`) | `verdict === "match"` (enum match/mismatch/inconclusive) |
  | `evals/models.docs.eval.ts` | docs | add model `openai/fixture-chat` to provider, reload | `StructuredOutputJudge` strict, `allowExtras:false` over `inspectAddedModel` |
  | `evals/openai-provider.docs.eval.ts` | docs | configure OpenAI-compatible "Acme" provider → local fixture server | StructuredOutputJudge over `inspectProvider` with **live probe completion** |
  | `evals/custom-provider.docs.eval.ts` | docs | implement NDJSON streaming provider from JSON API-doc fixture | StructuredOutputJudge + live probe |
  | `evals/extensions.docs.eval.ts` | docs | create `hello` tool extension, reload, call with "Bob" | StructuredOutputJudge + `ToolCallJudge({expectedTools:[{name:"hello",arguments:{name:"Bob"}}]})` |
  | `evals/tui.docs.eval.ts` | docs | replace footer `42.2%/272k (auto)` with 10-cell progress bar | custom `ContextFooterJudge` rendering real `InteractiveMode` into 100×30 `RecordingTerminal` at 42.2/65/120%; Levenshtein only metadata |
- Fixtures: `evals/acme-server.ts` local HTTP on `127.0.0.1:0`, checks bearer/`x-acme-key`, `stream:true`, model id; `validRequestReceived` only on exact probe prompt (`:65-136`); `evals/configured-runtime.ts` rebuilds `ModelRuntime` with `allowNetwork:false`, runs `completeSimple`, checks built-ins preserved vs pristine runtime (`:69-119`).
- **Agent under test** = real coding agent in-process via `createAgentSessionServices` + `createAgentSessionFromServices`, `thinkingLevel:"off"` (`harness.ts:337-355`) → [[pi--sdk-embedding]]. Model from option or `PI_PROVIDER`+`PI_MODEL` (both required); one model per invocation; plan identity `provider/model` (`harness.ts:70-80`; `cli.ts:27-117`).
- **Credentials**: host `auth.json` → `InMemoryCredentialStore`; inside sandbox delete `auth.json` and credential env var after resolving auth, restore after (`harness.ts:312-335,443-445`).
- **Per-run isolation**: fresh `mkdtemp` root with `workspace/`, `home/`, `home/.pi/agent`; overrides `HOME`, `USERPROFILE`, `PI_CODING_AGENT_DIR`; strips `PI_EVAL_*`; fails if any extension other than the inline prompt transform loaded; root removed in `finally` (`harness.ts:82-100,283-286,357-363,437-441`).
- **Docker (docs evals)**: multi-stage `node:24-slim`; builder `npm ci` + `build:offline` + `install-runtime.mjs` installing packed local workspace tarballs, stripping docs/examples/src/test from other internal packages symmetrically (`docker/Dockerfile:1-10`; `install-runtime.mjs:16-53`); `without-docs-install` deletes coding-agent README/CHANGELOG/docs/examples (`Dockerfile:14-19`); evaluator sources `chmod go-rwx` (`:27-35`); targets set `PI_EVAL_VARIANT` (`:38-44`). `docker run --rm --read-only`, tmpfs `/tmp` + `.vite-temp`, `/artifacts` bind, UID/GID 65532, auth.json read-only at `/run/pi-eval-secrets/auth.json` (`src/docker.ts:98-122`). Entrypoint asserts `/repo` allowlist, no internal docs visible, variant docs presence/absence, root-owned evaluator files mode `&077==0`, probes from a UID-65532 child that eval sources are unreadable, coding-agent resolves from `dist/index.js`; runs `vitest list`/`run` with `vitest-evals/reporter` + JSON (`docker/entrypoint.ts:27-143`).
- **Privilege drop** `enterToolSandbox`: must start as root; chmod Vite SSR transform caches 0600; chown run root; `setgroups([])`, `setgid`, `setuid(65532)`; verify it CANNOT read protected transformed modules (`harness.ts:130-187`). Docs harness refuses outside container (`PI_EVAL_CONTAINER=1` + sandbox identity, `:526-528`).
- Docs eval tools default `read, write, edit, grep, find, ls` — no bash/web (`harness.ts:484-485`); network stays open for provider traffic, so Docker can't prove agent code never uses network (`packages/evals/README.md:90`).
- **Treatment** = docs on disk + `<docs>` system-prompt section; `without_docs` strips `<docs>…</docs>` via hidden inline extension on `before_agent_start` (`harness.ts:289-301,494-506,534`); `verifySystemPrompt` fails closed if `<rules>` missing or `<docs>` presence ≠ variant (`:257-270`); transform records what it SENT (`:288-298,388-390`).
- Turn validity: error if last `stopReason` ∉ {stop, toolUse} or `stop` with empty text (`harness.ts:238-255`); abort → `session.abort()`.
- **Orchestration** (`src/cli.ts`): flags `--provider`, `--model`, `--runs-per-variant`, `-t`; others throw; runs default 1 or `PI_EVAL_RUNS_PER_VARIANT` (`:24-83`) → build both images, require distinct IDs (`:131-133`; tag `pi-evals-<sha256(repoRoot)[:12]>`, `docker.ts:39`) → discover cases in both, require identical cohorts (`:106-148`) → `createTaskPlan` alternates order: odd runs without→with, even with→without "to reduce order bias" (`plan.ts:51-56`; `packages/evals/README.md:80`) → `protocol.json` (schemaVersion 1, SHA-256 `protocolDigest`) + `expected-runs.json` (`cli.ts:153-168`) → fresh container per task, exact escaped name regex (`docker.ts:161-175`); missing report → `errored`, cohort continues; `observations.jsonl` rewritten per task (`cli.ts:175-183`) → `report.json`/`report.txt`, **exit 1 if any blocked pair** (`:186-192`). Artifacts `.eval/<ISO>_<uuid>/` mode 0700, gitignored; native `session.jsonl` per arm (`piSessionJsonl`) (`report.ts:9,122-134`; `harness.ts:420-431`).
- **Graders**: all deterministic, no LLM-as-judge; state-based oracles reload runtime from what the agent wrote (`configured-runtime.ts:65-93`); `judgeThreshold:null` — low score is data; vitest assertions reserved for broken suite invariants (`packages/evals/README.md:129`).
- **Metrics/report**: per run `inputTokens, outputTokens, totalTokens, toolCalls, cacheReadTokens, cacheWriteTokens, estimatedCostUsd` (only if model has non-zero pricing), `timings.totalMs`, `systemPromptSha256` (`harness.ts:391-412,468`). Outcomes `scored|unscored|skipped|pending|errored`; vitest `failed` → `errored` (`report.ts:32-34,101-106`). Errored if report ≠ exactly one assertion+case, no harness run, **reported provider/model ≠ planned**, negative/non-finite metric, run errors, score ∉ [0,1] (`report.ts:85-180`). Pass = score ≥ 1. Report schemaVersion 3: `controlPassRate`, `treatmentPassRate`, `lift`, paired `meanDelta` tokens/tools/latency/cost, `operationalTotals` + `availableRuns` (`report.ts:43-73,359-413`). **Fail-closed pairing**: pair blocked unless each variant expected once, observed once, scored; any blocked pair → headline rates + lift `null` ("withheld"); missing metrics never 0 (`report.ts:249-318,378-447`). Flags `no-lift`, `negative-delta`, `control-saturated`, `treatment-saturated`, `flaky` (`report.ts:336-357`).
- Run: `npm run eval` → `eval:host && eval:docs`; `npm test -w packages/evals` (runner unit tests) is the only CI part — **model evals not in CI** (`.github/workflows/ci.yml:45`; grep).

### Conformance / differential / benchmark suites (pluggable backends)
- `@earendil-works/pi-durable/testing`: runner-independent cases + adapter for Vitest/Jest `expect` (`packages/durable/src/testing/assertions.ts:16-39`; `packages/durable/src/testing/runner.ts:12-38`).
  - `createStorageConformance` (`storage-conformance.ts:94-1661`): `withStorage(use)` once per case; root conversation ID 1, `mintId` from 2; atomic multi-table commit with rollback and monotonic seq; detached records, `__proto__` keys safe; out-of-order entry indexing; cursor pagination; deep fork history; either-order table scans (`4dd2af42c`); task/submission/document invariants (rewindable reconstruction, version transitions, index rollback); one global ID namespace, `MAX_SAFE_INTEGER` → sticky "ID space is exhausted"; all ops rejected after close. Consumers: Memory, JSONL (+reopen), SQLite (+reopen), SQLite on Cloudflare DO (`test/*-storage.test.ts`).
  - `createEnvConformance` (`env-conformance.ts`): fresh writable cwd; shell default `["sh","-c"]`; binary reader, dir reader, 9+ watch scenarios (30 000 ms per watch case, steps wait ≤3 s, `:107-108`), exec argv/stream tagging/exit codes, windowed exec exact tail + skip counts (`maxBytes:200, maxLines:5`, 2000 lines), timeout vs abort codes (`:100-466`). Consumers: durable Node env, `packages/env/test/conformance.test.ts:16`, `ssh-external.test.ts:74` (same suite over SSH) → [[pi--remote-execution-env]].
  - `createTelemetryAdapterConformance` → [[pi--install-telemetry]].
- Differential: `packages/env/test/differential.test.ts` (random op sequences, Remote vs Node); `packages/durable/test/tools-read-differential.test.ts` keeps pre-bounded `referenceRead` as oracle, 400 seeded files × 4 (offset,limit) trials incl. BOM, CRLF, invalid UTF-8, split multibyte, >`DEFAULT_MAX_BYTES` lines, NaN/−4/2.5/1e20 (`:13-178`).
- Benchmarks: exported `storage-benchmark.ts` workloads (scales 1k/10k; timing 1000 entries/300 tasks/300 docs; `REPLAY_TAILS=[0,16,128,1024]`, `HISTORY_SEGMENT_LENGTH=128`, `FORK_DEPTH=8`, `ENTRIES_PER_FORK=32`, `BATCH_SIZE=100`), each read with `expected(dataset)` oracle validated BEFORE timing; writes from pre-built fresh fixture pools (`storage-benchmark.ts:24-455`; `test/storage.bench.ts:21-185`). `tool-output-bench.ts` runs each scenario in its own child process for isolated peak RSS (1 GiB firehose, 30 s rounds, `bash cat 1 GiB`) with commit p50/p99, replayMs, peakRssMiB (`:4-125`).

## Constants
| name | value | path:line |
|---|---|---|
| `DOCUMENTATION_EVAL_TOOLS` | read, write, edit, grep, find, ls | `packages/evals/src/harness.ts:485` |
| sandbox UID/GID | 65532 | `packages/evals/src/docker.ts:110-111` |
| test/hook timeouts | 300 000 ms | `packages/evals/vitest.evals.config.ts:18-19` |
| thinking level | `"off"` | `harness.ts:350` |
| runs per variant default | 1 | `cli.ts:80` |
| report / protocol schemaVersion | 3 / 1 | `report.ts:76`, `cli.ts:154` |
| TUI fixture `CONTEXT_WINDOW` | 272 000 | `evals/tui.docs.eval.ts:16` |
| audit explanation maxLength | 2000 | `evals/documentation-audit.eval.ts:18` |
| env watch-case timeout | 30 000 ms | `durable/src/testing/env-conformance.ts:107-108` |

## Evolution
- 2026-07-25 `eafe11fb9` first vitest eval harness (#7085: `general-knowledge.eval.ts`, `pi-harness.ts`, `run-evals.mjs`); `6173017a7` dropped third-party `harness-pi-ai` (registry self-install broke releases), rebuilt on `createHarness`; `73c1696d9` pi-harness ~2100 lines → small adapter.
- 2026-07-26 `a0ac81c0a` invariants (`assert`) vs scored expectations (`expect`).
- 2026-07-27 `8a9763245`, `95d1c60fe`, `b56a35f90`, `b0864a66c` scenarios in cases, multi-step prompt/reload input, typed output, diagnostics preserved on cleanup failure.
- 2026-07-30 `32f3a9728` comparative harness (baseline/candidate, pass-rate lift + paired deltas); `6a3eae8e4` inputs normalized before hashing; `743e1595c` removed custom seeded shuffle (`seed: 42`).
- 2026-09-07 `9211da172` documentation audit evals (#9280); 2026-09-09 `f3564a1d4` docs manifest + nav reachability; 2026-09-11 `b21588402` customization doc evals + repeatable reports (#9491).
- 2026-09-15 `d7296c063` Docker-isolated documentation-lift evals (#9635): Dockerfile/entrypoint, cli/plan/report, fail-closed pairing; deleted comparative layer + `run-evals.mjs`.
- 2026-09-16 `1247476e6` XML `<rules>/<docs>/<cwd>` markers; 2026-09-17 `42cd371ba` TUI footer eval (#9705), `5a3a03a7f` prompt validated from transcript + partial usage preserved (#9706), `16292398a` validate the prompt the transform SENT; 2026-09-22 `25cc5c7bf` audit prompt adjusted.
- 2026-09-23/24 `b45597504` exported scoped storage conformance (#9977), `3f0573877` benchmark workloads exported.
- 2026-10-04 `4748c627a`… env conformance cases; `acfc60198` `sleep 5`→`sleep 2`, wait for killed Windows commands; `be882f3fa` 30 s watch-case timeouts; 2026-10-05 `cd60a5b99`; 2026-10-06 `4dd2af42c` either-order scans.

## Evidence commits
`eafe11fb9` `6173017a7` `73c1696d9` `a0ac81c0a` `b0864a66c` `32f3a9728` `743e1595c` `9211da172` `b21588402` `d7296c063` `1247476e6` `42cd371ba` `5a3a03a7f` `16292398a` `b45597504` `3f0573877` `acfc60198` `be882f3fa` `4dd2af42c`

## Quirks
- Documentation-audit uses the model under test as auditor, runs on host without Docker, repo-wide read access — CI docs-lint or manual only? (open).
- Network egress open: an agent in `without_docs` could fetch docs from the web via a self-written extension (only bash/web tools excluded) — contamination risk (open).
- Lift numbers are not published anywhere; no CI job runs model evals.
- Scripted fake provider exists for tests, not evals: pi-ai `fauxProvider()` (`packages/ai/src/providers/faux.ts`, added `ef6af5ebb`) — scripted queue of `AssistantMessage`s/factories, exhausted queue → "No more faux responses queued" (`:530-541`), random 3–5-token chunking with optional `tokensPerSecond`, abort between chunks, **simulated per-`sessionId` prompt cache** from common prefix (`:245-285`), deferred responses (`:543-661`), default model `faux-1` 128k/16384 (`:24-28,480-505`); used by `packages/agent/test/e2e.test.ts`, `packages/ai/test/faux-provider.test.ts`. Model evals instead use real models + local fixture servers (acme-server).

## Failures
[[eval-harness-installs-registry-copy]] · [[eval-assertions-conflated-with-scores]] · [[failed-run-diagnostics-lost]] · [[bespoke-shuffle-order-bias]] · [[treatment-transform-marker-drift]] · [[validation-reads-projected-not-sent-prompt]] · [[control-arm-docs-leakage]] · [[eval-judges-readable-by-agent]] · [[conformance-suite-platform-timing]] · [[rewrite-drops-test-coverage]]
