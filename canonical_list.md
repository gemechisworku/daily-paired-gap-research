# Canonical List: Papers, Tools, and Patterns for FDE Work

This is my annotated contribution to the cohort canon: resources and patterns that changed concrete engineering decisions in Week 12. I selected items by one rule only: if a source or method did not change what I would build, measure, or ship, it is not in this list.

## Papers and Core References

## 1) Cohen (1960) and the kappa lineage
- Why it matters: Gives the base definition of chance-corrected agreement, still widely cited in evaluation reviews.
- When to use: Any rubric or labeling report that includes inter-rater reliability.
- Practical caution: Kappa alone is not enough under skewed marginals.

## 2) Feinstein and Cicchetti (1990), "High agreement but low kappa"
- Why it matters: Canonical explanation of the paradox most teams hit when one class dominates.
- When to use: Defending high raw agreement with low kappa in audit-facing docs.
- Practical impact: Prevents false "rubric failure" conclusions from single-metric reading.

## 3) Hu et al. (LoRA, 2021)
- Why it matters: Foundational mechanics for low-rank adaptation (`W' = W + BA`) and intrinsic-rank intuition.
- When to use: Adapter design reviews, especially when tradeoffs are hidden behind defaults.
- Practical impact: Forces explicit capacity allocation thinking, not default inheritance.

## 4) DistServe prefill/decode materials
- Why it matters: Strong system-level framing of inference as two distinct phases with different bottlenecks.
- When to use: Investigating latency regressions in long-context generation.
- Practical impact: Stops "one latency number" anti-pattern; enables targeted optimization.

## 5) Transactional Outbox (microservices.io)
- Why it matters: Durable pattern for coupling intent persistence with external side effects under failure.
- When to use: Any workflow that must survive retries/timeouts without duplicate writes.
- Practical impact: Moves reliability from hopeful retries to explicit replay semantics.

## 6) Durable execution references (Temporal, Restate)
- Why it matters: Establishes runtime-level guarantees for long-running, failure-prone workflows.
- When to use: Multi-step orchestrations with external dependencies and strict correctness requirements.
- Practical impact: Clarifies when state-machine code is not enough and workflow runtime is justified.

## 7) PEFT documentation (Hugging Face)
- Why it matters: Practical implementation details for LoRA target modules, scaling, and adapter behavior.
- When to use: Fine-tuning setup reviews and adapter debugging.
- Practical impact: Bridges theory to concrete training configuration choices.

## 8) LLM inference benchmarking guidance (DigitalOcean article)
- Why it matters: Useful practical metric framing (TTFT, TPOT, E2E) for production inference conversations.
- When to use: Communicating performance findings to mixed technical audiences.
- Practical impact: Aligns teams on comparable, meaningful latency metrics.

## Tools and Instrumentation

## 1) Hugging Face `TextStreamer` first-token timing pattern
- What it does: Captures first-token timestamp to approximate prefill end (TTFT) and separate decode timing (TPOT).
- Why canonical: Minimal code change, high explanatory power.
- Best use: Prompt-length sweeps, regression triage.

## 2) `torch.profiler` + Chrome trace (`export_chrome_trace`)
- What it does: Shows kernel-level execution phases and validates prefill/decode attribution.
- Why canonical: Converts assumptions into inspectable kernel evidence.
- Best use: Deep performance root-cause analysis.

## 3) Deterministic idempotency key generator
- What it does: Produces stable operation keys (`entity:op:payload_hash`) across retries.
- Why canonical: Core primitive for duplicate-side-effect prevention.
- Best use: External writes in REST and tool-mediated backends.

## 4) Operation-state checkpoint ledger
- What it does: Tracks operation lifecycle (`PENDING`, `IN_FLIGHT`, `COMMITTED`, `FAILED`) for resume-safe retries.
- Why canonical: Defines commit boundaries and replay behavior explicitly.
- Best use: Multi-step workflows with partial failure risk.

## 5) LoRA adapter diagnostics (`print_trainable_parameters`, per-module norm checks)
- What it does: Audits whether capacity allocation is meaningful or diffuse.
- Why canonical: Detects hidden configuration debt early.
- Best use: Small-dataset preference tuning and module ablation planning.

## 6) Paired-bootstrap evaluation helper
- What it does: Provides uncertainty-aware delta estimates between variants.
- Why canonical: Prevents over-claiming from point estimates.
- Best use: Held-out benchmark comparisons and honest model-card reporting.

## 7) Reliability metric bundle templates
- What it does: Forces reporting of agreement + chance-corrected metric + marginals + confusion matrix.
- Why canonical: Reduces misinterpretation in rubric reliability claims.
- Best use: Annotation and evaluation documentation.

## Patterns Worth Reuse Across Projects

## Pattern A) Mechanism -> Metric -> Decision
- Definition: Every claim must map from a causal mechanism to observable signals, then to a decision.
- Why it matters: Prevents hand-wavy optimization and post-hoc narratives.
- Week 12 example: TTFT/TPOT split leading to prefix-caching decision.

## Pattern B) Commit is a local durability event
- Definition: External success is tentative until local committed state is durably written.
- Why it matters: Clean boundary for retries and compensation.
- Week 12 example: Resume from first uncommitted booking operation.

## Pattern C) Ownership split in hybrid agents
- Definition: Keep irreversible transitions deterministic; keep semantic ambiguity model-mediated; harden boundary contracts.
- Why it matters: Most failures happen at handoff, not inside one layer.
- Week 12 example: REST vs MCP idempotency responsibility mapping.

## Pattern D) Capacity is allocated, not defaulted
- Definition: LoRA rank/module choices are architecture decisions tied to task and data.
- Why it matters: Defaults can hide overfit and poor transfer.
- Week 12 example: replacing all-module spread with focused module targeting.

## Pattern E) Report metric semantics, not only values
- Definition: Explicitly state what a metric means in the deployment context.
- Why it matters: Avoids capability vs reliability confusion.
- Week 12 example: pass^1 deployment truth vs pass^k capability ceiling.

## Pattern F) Use metric systems for reliability claims
- Definition: Never publish a single reliability number without its context metrics.
- Why it matters: Prevents paradox-driven misreads.
- Week 12 example: raw agreement + kappa + marginals + confusion matrix (+ AC1 when imbalanced).

## What Other FDEs Should Read First

If an FDE has one hour, this is the highest-yield order:
1. DistServe prefill/decode overview.
2. Transactional outbox + durable execution intro.
3. LoRA paper section on intrinsic rank plus PEFT practical docs.
4. Feinstein and Cicchetti on kappa paradox.

If an FDE has one day, implement one instrumentation from each domain:
1. Add TTFT/TPOT logging.
2. Add operation commit ledger + idempotency keys.
3. Add one LoRA module/rank ablation.
4. Add a reliability metric bundle table to evaluation docs.

That sequence creates immediate leverage in performance, reliability, modeling, and evaluation quality.
