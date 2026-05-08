# Day 4 — Evaluation & statistics: diagnostic questions

Five questions about pass^k vs pass^1, paired-bootstrap CIs, LLM-as-a-judge biases, and the limits of n-gram + bag-of-tokens contamination checks. Each names a specific gap, points at a shipped artifact in this repo it would let me revise, and is scoped for a single 600–1,000-word explainer.

---

### Question: My held-out evaluation is structurally pass^1 with `n=50` and one sample per task — what does pass^k actually measure that pass^1 hides, and which of my held-out failures would each metric label "real"?

`scripts/run_act4_ablations.py` `summarize_variant` computes `pass_rate` from a single per-task boolean `score["pass"]` — i.e. each variant produces one candidate per held-out task and the run is sealed at that point (`ablations/ablation_results.json` `variant_summary.trained.pass_rate: 0.82`, `n=50`). I have nothing in the harness that re-samples k candidates per task at temperature > 0, so what I am reporting is pass^1 even though I describe it as a "pass rate."

The diagnostic gap is **what the difference between pass^1 and pass^k actually buys you on an agent-output benchmark like this** — not on HumanEval where pass^k has a clean coding-problem definition. Concretely:

1. For my **trained variant** at `pass^1 = 0.82`, the 9 missed tasks could be *deterministic* failures (the same 9 tasks miss every time at temperature 0) or *stochastic* near-misses (a different ~9 of 50 missed each draw at temperature > 0). pass^k at `k=5` separates these, but in opposite directions: a deterministic-miss model has pass^k = pass^1; a stochastic-miss model has pass^k >> pass^1.
2. For a **deployment decision**, which of the two metrics is the right one — and when does pass^k > pass^1 *mislead* you about production reliability (e.g., when the production loop only runs the model once per turn, so the `k>1` recovery never happens)?
3. **What harness change** is required to compute pass^k on this benchmark — multi-sample at `temperature > 0` per task, or candidate-pool generation with a separate selector, or both — and how does that change the meaning of `paired_bootstrap` over the resulting per-task statistic?

Knowing this would let me revise the **"Pass rate" definitions in `model_card.md` §Results and `bench_challenge_report.md` §1**, both of which currently report a single number with no statement about `k=1` vs `k>1`, and would let me write a precise "Pass-rate measurement" line in **`tenacious_bench_v0.1/datasheet.md` §4 (Labeling)** so a downstream user does not silently confuse pass^1 with pass^k.

Generalizable because every agent-benchmark scoreboard reports a "pass rate" without saying which `k`, and that ambiguity is exactly where deployability claims and capability-ceiling claims get conflated.

---