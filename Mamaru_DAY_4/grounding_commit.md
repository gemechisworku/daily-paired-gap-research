# Day — Grounding Commit

**Asker:** Mamaru Yirga

**Artifact edited:** `docs/inter_rater_agreement.md`

**Commit:** [`f9ae0a8`](https://github.com/mamee13/sales-agent-eval-bench/commit/f9ae0a8b811e112eed5548a19286c6bcb6a9da3c)

**What changed and why:**

The Key Finding section previously concluded: "The revised rubric meets the ≥80% agreement threshold
across all dimension categories" — and cited the 94.9% overall agreement — without explaining why
`tone_quality`'s kappa=0.000 does not undermine that conclusion. A reviewer reading the table would see
kappa=0.000 next to 88.9% agreement and have no explanation for the apparent contradiction.

After reading the explainer on the Kappa Paradox, I added the following paragraph to the Key Finding
section immediately after the existing conclusion:

---

**Added text in `docs/inter_rater_agreement.md`:**

> **Note on `tone_quality` kappa=0.000 at 88.9% agreement (the Kappa Paradox):** This result is a
> known statistical artifact, not a reliability failure. Cohen's kappa measures agreement beyond what
> would be expected by chance given each rater's marginal label distribution. For `tone_quality` (9
> tasks), both labeling rounds assigned "pass" to 8 of 9 items — a heavily imbalanced marginal. When
> one label dominates, the expected chance agreement (p_e) rises to match the observed agreement (p_o),
> driving the kappa numerator (p_o − p_e) toward zero regardless of how consistently the rubric was
> applied. The 0.000 kappa reflects label prevalence, not annotation inconsistency. Raw agreement
> (88.9%) is the appropriate primary metric here because the rubric's goal is operational consistency
> — confirming the same deterministic criteria fire the same way across labeling rounds — not
> discriminative performance on a balanced dataset. For future versions with more `tone_quality` tasks,
> Gwet's AC1 should be reported alongside raw agreement as it is robust to the prevalence effect and
> will not produce a paradoxical zero under imbalanced labels. See: Feinstein & Cicchetti (1990),
> *High agreement but low kappa: I. The problems of two paradoxes*, Journal of Clinical Epidemiology.

---

**Why this change matters:** The Key Finding section is the section a hiring manager or external
reviewer reads first when auditing the benchmark's reliability claims. Before this edit, the section
contained an unexplained contradiction. After this edit, the contradiction is named, mechanically
explained, and defended with a citation. The rubric's reliability conclusion is now defensible.
