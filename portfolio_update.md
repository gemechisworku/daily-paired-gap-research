# Portfolio Update (Weeks 10-11): Reliability and Evaluation Maturity Through Four Grounding Commits

This week produced four grounding commits that materially improve the credibility of my Weeks 10 and 11 portfolio. For an FDE hiring review, the key point is not that I wrote more documentation; it is that I converted ambiguous claims into auditable engineering decisions tied to concrete artifacts.

## Executive Summary

Before these commits, the portfolio already showed strong implementation momentum in agent orchestration, model adaptation, and benchmark reporting. The main gap was defensibility under failure and review pressure: several claims were directionally right but not yet specified at the mechanism level.

After these commits, each major claim now has:
- an explicit boundary condition,
- a measurable interpretation,
- and a reproducible action policy.

That shift raises the portfolio from "working prototype" quality toward production-minded FDE quality.

## Grounding Commit 1 (Day 1): Prompt-constraint reliability in long-context JSON generation

- Artifact: `conversion-engine` `agent/services/enrichment/llm.py` and related test coverage.
- Commit reference: `d22003e02359ce8b4c77b793363e6ed2cf12fc78`.
- Improvement: Introduced a shared constraints constant and reinforced it both in system scope and at the tail of the user JSON payload (constraint sandwiching).

Why this improves Weeks 10/11 narrative:
- Moves from generic "prompt hardening" language to a specific transcript-layout intervention.
- Shows awareness that compliance failures are often context-position failures, not only model failures.
- Adds a test-backed reliability behavior in code, not just a recommendation in prose.

Hiring-manager signal:
- I can identify subtle instruction-following failure modes and land a concrete, low-risk mitigation with clear tradeoff (slight token overhead for better constraint adherence).

## Grounding Commit 2 (Day 2): Commit-point and idempotency ownership clarity for retries

- Artifact: `conversion_engine/README.md` (architecture and reliability semantics for dual backend execution).
- Improvement: Defined commit point as durable local persistence after successful side effect; documented retry-resume rule and REST vs MCP idempotency ownership split.

Why this improves Weeks 10/11 narrative:
- Upgrades "supports retries" to "retry-safe by design" with explicit semantics.
- Clarifies how duplicate side effects are prevented under ambiguous timeout failures.
- Demonstrates architecture thinking across transport modes rather than single-path assumptions.

Hiring-manager signal:
- I understand distributed failure semantics and can document reliability contracts clearly enough for peers to implement consistently.

## Grounding Commit 3 (Day 3): Honest adapter-configuration rationale in model artifacts

- Artifacts: `model_card.md` and `methodology_rationale.md`.
- Commit reference: `b7c14d5`.
- Improvement: Added explicit critique of inherited LoRA defaults (r=16 across seven modules for 306 pairs), explained likely gradient diffusion/shortcut risk, and documented a more defensible alternative configuration.

Why this improves Weeks 10/11 narrative:
- Replaces silent default usage with named engineering tradeoffs.
- Aligns strong validation results with weaker held-out reranking through mechanism-level explanation.
- Strengthens model-card honesty by documenting limitations as design debt, not as vague caveats.

Hiring-manager signal:
- I can diagnose where a training setup underperforms in transfer and rewrite artifact language to remain technically honest under scrutiny.

## Grounding Commit 4 (Day 4): Statistical reliability interpretation under class imbalance

- Artifact: `docs/inter_rater_agreement.md`.
- Commit reference: `f9ae0a8`.
- Improvement: Added explicit explanation of the Kappa Paradox for `tone_quality` (high raw agreement with near-zero kappa under skewed marginals), with reporting guidance and future AC1 recommendation.

Why this improves Weeks 10/11 narrative:
- Removes an unresolved contradiction in benchmark reliability claims.
- Demonstrates ability to defend evaluation methodology mathematically, not rhetorically.
- Improves trustworthiness of rubric conclusions for external reviewers.

Hiring-manager signal:
- I can identify when a metric is being misread and correct the reporting policy so product decisions are not driven by statistical artifacts.

## Collective Impact on Portfolio Quality

Together, these four commits improve portfolio quality on four dimensions that matter in FDE roles:

1. Production reliability
- Clearer retry and side-effect semantics reduce operational risk in orchestrated workflows.

2. Measurement rigor
- Better metric decomposition (latency, pass metrics, reliability metrics) prevents false optimization targets.

3. Technical honesty
- Model-card and methodology updates explicitly name what is known, what is uncertain, and why.

4. Communication under review
- Artifacts are now easier for cross-functional stakeholders to audit because assumptions and tradeoffs are explicit.

## Why This Matters for an FDE Hiring Decision

FDE work is evaluated on whether an engineer can translate ambiguous model behavior into stable product decisions under real constraints. These four commits show that I can:
- move from concept to mechanism,
- instrument and define the right boundary conditions,
- and leave behind artifacts that another engineer can operate and defend.

The result is a materially stronger Weeks 10 and 11 portfolio: not only functional, but review-ready for reliability, evaluation integrity, and operational clarity.
