---
type: failure
concepts: [tool-output-truncation, per-model-system-prompt, minimal-system-prompt]
harnesses: [codex]
---
**Symptom** — The prompt said "Read files in chunks with a max chunk size of 250 lines… Command line output will be truncated after 10 kilobytes or 256 lines of output, regardless of the command used." (added `90d892f4fd` 2025-08-12). After `tool_output_token_limit` became configurable and defaults were raised ~4× for codex models, the prompt still said 10 KB and the model self-limited to the stale number — issue #7867: "the system prompt for gpt-5.1 (non-codex) models hard-codes outmoded output limits via prompting"; #7906: "Raising the max token limit doesn't help if the system prompt still tells the agent outputs are capped at 10KB… When I remove that line… the model does not start spewing huge outputs".

**Root cause** — A configurable harness constant restated as a literal in static, model-owned prompt text.

**Fix · [[codex]]** — `570eb5fe78` 2025-12-12 "chore(prompt) Remove truncation details (#7941)" and `26d0d822a2` same day for the base prompt; line reduced to "Do not use python scripts to attempt to output larger chunks of a file." (`codex-rs/protocol/src/prompts/base_instructions/default.md:265`).

**Lesson** — Never restate a configurable harness limit as a literal in static prompt text; interpolate it from the constant (pi's tool descriptions do) or omit it.

Related: [[tool-output-truncation]] · [[per-model-system-prompt]] · [[minimal-system-prompt]] · [[prompt-names-unavailable-tools]] · [[prompt-rewrite-drops-load-bearing-lines]] · [[codex--tool-output-truncation|codex]]
