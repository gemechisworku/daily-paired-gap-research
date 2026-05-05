1/6
Why did my LLM get slower when I added more context... even though output length stayed the same? ??

I dug into Hugging Face `generate()` to answer one practical question:
Where does the extra latency actually come from?

2/6
Most teams measure one number: end-to-end latency.
That hides the real bottleneck.

If you run long-context tasks (like pairwise preference scoring), you need to split latency into phases or you�ll optimize the wrong thing.

3/6
`generate()` is really 2 phases:

- Prefill: model processes the full prompt once, builds KV cache, then emits first token
- Decode: model generates the rest token-by-token using that cache

Mapping:
TTFT � prefill
TPOT � decode

4/6
How to measure this in practice:

- Start timer before `model.generate()`
- Capture time when first token is emitted (streamer callback)
- Stop timer at generation end

Now you can compute `ttft`, `tpot`, and `e2e` per request.

5/6
What happens when prompt tokens increase (with fixed completion length)?

Usually:
- `ttft` jumps the most
- `tpot` increases slightly or stays near-flat

Meaning: slowdown is mostly prefill cost, not decode throughput.

6/6
How to prove it + fix it:

Log `prompt_tokens`, `completion_tokens`, `ttft_ms`, `tpot_ms`, `e2e_ms` across multiple prompt lengths.
If TTFT grows much faster than TPOT, prefill is your bottleneck.

Best fix for shared prompts: prefix caching.
Compute shared-prefix KV once, reuse it across requests. ??
