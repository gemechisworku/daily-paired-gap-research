# Day — Sign-Off

**Asker:** Mamaru Yirga

**Gap closure status:** closed

**What I understand now that I didn't before:**

Before reading this explainer, I had a contradiction sitting in `docs/inter_rater_agreement.md` that I
could not explain: `tone_quality` showed 88.9% raw agreement between my two labeling rounds but a
Cohen's kappa of 0.000, and I concluded the rubric "passes" the ≥80% threshold without being able to
defend that conclusion against a reviewer who asked why I was ignoring kappa.

The explainer closed that gap by naming the mechanism: the Kappa Paradox. When one label dominates the
marginal distribution — in my case, both labeling rounds assigned "pass" to 8 of 9 `tone_quality` tasks
— the expected chance agreement (p_e) rises to match the observed agreement (p_o), collapsing the
numerator of the kappa formula to near zero. The 0.000 is not evidence that my rubric is unreliable; it
is a mathematical artifact of the label imbalance, not a signal about annotation quality.

What changed concretely: I now understand that raw agreement is the correct primary metric for my
rubric because my goal is operational consistency — confirming that the same rubric criteria produce the
same label when applied twice — not discriminative performance on a balanced dataset. Kappa is the right
metric when you need to know whether raters are making nuanced distinctions above chance; it is the
wrong metric when the task is to verify that a deterministic rubric fires consistently on a
pass-dominated dimension. The explainer's summary table (Balanced Labels → use kappa; Imbalanced Labels
→ kappa paradox, raw agreement is more faithful) gives me the exact framing I need to defend the Key
Finding section of `docs/inter_rater_agreement.md` to a hostile reviewer.

I also now know what a stronger alternative looks like: Gwet's AC1 is robust to the prevalence effect
and would give a non-zero reliability estimate even under label imbalance. If I add more tasks to
`tone_quality` in v0.2, I should report AC1 alongside raw agreement to preempt the kappa objection
entirely.
