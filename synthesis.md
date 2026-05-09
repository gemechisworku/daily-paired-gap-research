# Week 12 Synthesis: 8 Gaps Closed Across Inference, Reliability, and Evaluation

This week closed eight concrete engineering gaps: four that I explicitly named from my own project pressure points, and four that I researched to answer partner questions with mechanisms, diagnostics, and implementation choices. Across all four days, the pattern was consistent: every gap looked abstract at first, but became tractable once we forced a strict bridge from concept -> observable signal -> code or documentation change.

A useful way to summarize the week is that I moved from "I can describe the issue" to "I can defend an operational decision under review." The four grounding commits are the strongest proof of that move, because they did not just add content; they tightened reliability boundaries, clarified measurement semantics, and documented tradeoffs in a way a reviewer can audit.

## The 4 Gaps I Named

## 1) Constraint obedience in long-context chat layouts

I named a gap around message construction in `OpenRouterJSONClient.generate_model`: when the user payload is a large serialized JSON block, why do compact instruction lines near the front of context often have more influence than equivalent reminders buried late, and what should be changed in transcript layout? This mattered because the failure mode is subtle: output still "looks valid" until edge cases drift into markdown, schema breaks, or invented evidence.

The closure was not theoretical. I moved to a concrete "constraint sandwich" design: keep global rules in system scope, then repeat the same compact rules after the user JSON tail where the model begins generation. The grounded tradeoff was explicit: small token overhead in exchange for better local retrievability of constraints at generation time. This yielded a defensible prompt engineering rule: do not treat instruction placement as cosmetic in long noisy contexts.

## 2) Scaffolding-driven vs model-driven planning ownership

I named a second gap around planning boundaries in hybrid agent systems: which decisions should stay deterministic in orchestration logic, which should remain model-mediated, and where brittle handoffs appear under ambiguous user requests.

What became clear is that most production failures are boundary failures, not pure "LLM quality" failures. The right mental model is decision-class ownership: deterministic control for irreversible transitions and policy gates; model judgment for semantic disambiguation; and explicit interface checks where model output becomes executable state. That reframed architecture review from "how smart is the model" to "where do we allow uncertainty to become commitment."

## 3) DPO overoptimization without an explicit reward model

I named a third gap from an uncomfortable metric pattern: held-out mean score rising while pass rate drops on hard-honesty slices. The original intuition I needed to correct was: "DPO has no separate reward model, so reward overoptimization should not apply."

The gap closed once I adopted the implicit-reward view: DPO still optimizes a proxy through chosen/rejected log-prob ratios relative to a reference policy. That made the observed pattern mechanically plausible instead of contradictory. The key outcome was diagnostic discipline: distinguish proxy overoptimization from noise overfit and distribution shift by reading metric divergence shape, margin behavior, and slice-level failures together, then pick corrective dials (beta pressure, epoch budget, KL-aware variants) by signature rather than by guesswork.

## 4) pass^1 vs pass^k semantics in held-out evaluation

I named a fourth gap in evaluation reporting: my harness was structurally single-sample per task, so the reported pass rate was pass^1, but the language around results did not always state that explicitly.

The closure was to separate two claims that are often conflated:
- pass^1 is deployment-truth reliability for one-shot systems.
- pass^k is capability ceiling only if the product actually samples and selects multiple candidates.

This sounds obvious, but it changes interpretation of regressions and model comparisons. A model with modest pass^1 and strong pass^k might be highly recoverable in a rerank architecture, yet still risky in a single-shot production loop. Naming this boundary tightened how I will write model cards and benchmark datasheets going forward.

## The 4 Gaps I Researched

## 5) Prefill vs decode attribution in `generate()` latency

In the Day 1 explainer I researched how to decompose wall time into prefill and decode in a way that matches transformer internals and can be measured in real runs. The practical contribution was the TTFT/TPOT instrumentation pattern using first-token timing via a streamer callback, plus optional kernel-trace validation with `torch.profiler`.

Why this mattered to the cohort: it gives a repeatable method to prove where latency moved when prompt length changes. Instead of "it got slower," we can now show whether TTFT scales superlinearly while TPOT stays near-flat, then justify prefix-caching investment with evidence.

## 6) Commit points, idempotency, and retry-safe side effects

In the Day 2 explainer I researched reliability mechanics for orchestrated external writes (contact upsert -> note append -> booking linkage). The core contribution was a strict commit-point definition: commit is the durable local state write after successful side effect confirmation, not the remote API call boundary alone.

That distinction enables correct retry behavior: resume from first uncommitted operation, skip committed operations, and preserve deterministic idempotency keys across REST and MCP execution paths. This closed a common architecture gap where teams claim "supports retries" but cannot prove duplicate-side-effect safety under ambiguous timeouts.

## 7) LoRA configuration as capacity allocation, not default inheritance

In the Day 3 explainer I researched whether LoRA defaults were defensible for a small preference dataset. The useful contribution was a mechanistic split of module roles (attention projections for what to notice, MLP projections for how to score) and a practical warning about gradient diffusion when limited supervision is sprayed across all seven projection groups.

For the cohort, this added a stronger adapter-design heuristic: treat rank and target modules as an allocation problem linked to rubric complexity and data volume, and evaluate with held-out reranking behavior, not validation accuracy alone.

## 8) Inter-rater reliability under prevalence distortion (Kappa Paradox)

In the Day 4 explainer I researched why high raw agreement can coexist with near-zero Cohen's kappa under imbalanced marginals. The practical value was twofold: (1) a precise mathematical explanation of the apparent contradiction, and (2) a reporting policy that avoids single-metric failure.

This closed a critical documentation gap for rubric-based evaluation work. Instead of defending one number, we now defend a metric bundle: raw agreement, kappa, label distribution, confusion matrix, and an imbalance-robust alternative (for example, AC1) when prevalence is extreme.

## Most Surprising Thing I Learned

The most surprising result was how often "contradictions" were actually measurement-definition errors, not model behavior mysteries.

The cleanest example was Day 4: 88.9% agreement with kappa = 0.000 looks like either rubric failure or math failure until you inspect marginals and chance expectation. The same pattern appeared elsewhere in different forms:
- latency confusion disappears once TTFT and TPOT are separated,
- retry-safety confusion disappears once commit point is defined precisely,
- DPO confusion disappears once implicit reward and slice-level metrics are read together,
- pass-rate confusion disappears once k is stated and matched to deployment architecture.

The broader lesson is that evaluation and reliability improve fastest when we tighten definitions first, then optimize.

## Canonical Reading List I Contributed to the Cohort

I contributed a cross-domain reading set oriented around decisions we repeatedly face in production:

1. DistServe prefill/decode explainers for inference-phase reasoning and system-level scheduling intuition.
2. DigitalOcean's inference benchmarking writeup for practical TTFT/TPOT/E2E measurement framing.
3. Transactional Outbox references for durable side-effect orchestration and replay-safe pipelines.
4. Durable execution materials (Temporal/Restate) for long-running workflow correctness under retries.
5. LoRA foundation paper and PEFT implementation docs for low-rank adaptation mechanics and constraints.
6. Kappa paradox and prevalence literature for defensible interpretation of inter-rater reliability under imbalance.

These readings are canonical not because they are exhaustive, but because each one directly changes a real engineering decision in our week artifacts.

## Canonical Tool List I Contributed to the Cohort

I also contributed a concrete tool/pattern list that other FDEs can reuse immediately:

1. First-token timing instrumentation (`TextStreamer`) to separate prefill from decode without changing model internals.
2. `torch.profiler` + Chrome trace export for kernel-level validation of phase attribution.
3. Deterministic idempotency-key schema (`entity:operation:payload_hash`) for retry-safe external writes.
4. Operation-state checkpointing (`PENDING/IN_FLIGHT/COMMITTED/FAILED`) for resume-safe orchestration.
5. LoRA norm inspection and module-target ablations for adapter-capacity diagnosis.
6. Paired-bootstrap confidence intervals and explicit metric-slice reporting for honest benchmark claims.
7. Reliability reporting bundles (agreement + chance-corrected metric + label marginals + confusion matrix) for annotation audits.

## How the 8 Gaps Changed My Portfolio Narrative

Before this week, my Weeks 10 and 11 artifacts showed strong implementation energy, but some claims relied on implied assumptions. After this week, the claims are more audit-ready because each has an explicit measurement boundary:
- instruction robustness tied to transcript layout and tests,
- retries tied to commit semantics and idempotency ownership,
- model-quality claims tied to slice-level evaluation meaning,
- reliability claims tied to statistically coherent metric interpretation.

For an FDE audience, this matters because the job is not just model usage. It is translating ambiguous behavior into stable production decisions with traceable reasoning.

## Final Synthesis

This week was less about adding new buzzwords and more about increasing decision fidelity. The same method closed all eight gaps:

1. Name the ambiguity in operational terms.
2. Define the mechanism in falsifiable language.
3. Attach the mechanism to measurable signals.
4. Land the result in a durable artifact (code, test, model card, report, or README).

That loop is now reusable across inference performance, orchestration reliability, adapter tuning, and evaluation statistics. The biggest gain is not any single answer; it is a stronger habit of forcing architectural claims to survive contact with metrics and implementation boundaries.
