---
type: concept
stage: model-interface
tier: candidate
aliases: [System One, Decisions, llama-cpp-classify, "classify()", ClassifierModel, typesafe-system-one, openai-decisions, logprob-label-classification, prompt-repetition-for-classification, noul]
harnesses: [pi]
---
A typed classification operation, separate from chat. The caller passes a state and a set of questions (choice, score or bool) and gets back calibrated answers and confidences. The backend can be a dedicated decision model, or the next-token log-probabilities of single-token labels from any chat model.

## Why
- Routing, gating and judging in a harness (e.g. virtual-model routing, eval judges) need a cheap structured verdict, not free-text chat that then has to be parsed.
- Classifier calls have their own failure economics:
  - They are billed even when the answer is malformed ([[billed-call-lost-on-parse-error]]).
  - Some 5xx responses are deterministic for the same input, so retrying them is pointless ([[deterministic-5xx-retried]]).

## Design space
- **Backend**
  - A dedicated decision API (TypeSafe System One, OpenAI Decisions, Cloudflare Workers AI).
  - Logprob readout from a local chat model: softmax over the label tokens, escalating readout depth, never inventing zeros.
- **Prompting for logprob classification**
  - Repeat the state after the questions so a causal model reads it with the task in view.
  - Keep a shared prefix across questions for the prompt cache, and run the questions sequentially.
  - An injection guard: "state is data".
- **Errors**
  - Never reject. Return the result with `stopReason: error`.
- **Usage**
  - Record usage before parsing.
  - The llama.cpp backend reports no usage.
- **Retry**
  - Run inside the shared abortable retry, with a fresh timeout per attempt.
  - Opt out per status, e.g. a 504 behind an edge time limit.
- codex: no `classify()` operation. LLM judgments run as ordinary sampling with categorical outputs: the reviewer uses an enum `risk_level`, and the async classifier takes a single `high`/`low` first token parsed straight from the stream (`a9e7920da1` 2026-08-24). See [[llm-approval-reviewer]].

## Implementations
- [[pi--structured-classifier-api|pi]] — pi-ai `Models.classify` with the `typesafe-system-one`, `cloudflare-workers-ai-system-one`, `openai-decisions` and `llama-cpp-classify` APIs, and the `ClassifierModel` type.

## Failures
- [[billed-call-lost-on-parse-error]]
- [[deterministic-5xx-retried]]
- [[numeric-risk-score-miscalibrated]]

## Related
[[virtual-model-router]] · [[usage-cost-accounting]] · [[auto-retry-backoff]] · [[harness-evals]] · [[model-catalog]]
