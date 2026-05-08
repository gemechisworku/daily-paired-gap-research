## The Kappa Paradox: Why 89% Agreement Can Yield a Kappa of Zero

In the world of data annotation, evaluation rubrics, and inter-rater studies, **Cohen’s Kappa ((\kappa))** is often treated as a more “rigorous” alternative to raw agreement. Unlike simple agreement percentages, kappa attempts to correct for the possibility that raters might agree **purely by chance**.

However, practitioners frequently encounter a puzzling situation:

> **High agreement (e.g., 88.9%) paired with κ = 0.000**

This phenomenon is not a mistake—it is a well-documented statistical effect known as the **Kappa Paradox**. Understanding it is essential for correctly interpreting reliability metrics and defending methodological choices.

---

## 1. The Mechanics: How Kappa is Calculated

Cohen’s Kappa is defined as:

[
\kappa = \frac{p_o - p_e}{1 - p_e}
]

Where:

* **(p_o) (Observed Agreement):** The proportion of items on which raters agree.
* **(p_e) (Expected Agreement):** The proportion of agreement expected **by chance**, computed from the raters’ label distributions (marginals).

### Key Insight

Kappa does **not** measure agreement directly. Instead, it measures:

> **Agreement beyond what would be expected if raters were labeling randomly, given their individual label tendencies.**

---

### The Critical Condition

The paradox emerges when:

[
p_o \approx p_e
]

In this case:

[
\kappa = \frac{p_o - p_e}{1 - p_e} \approx 0
]

This means:

* Even **very high agreement** can yield **κ = 0**
* Because the agreement is **no better than chance**, given the label distributions

---

## 2. The Role of Marginal Distributions

### What Are Marginal Distributions?

Marginal distributions describe how frequently each rater uses each label.

For example:

| Label    | Rater A | Rater B |
| -------- | ------: | ------: |
| Positive |     90% |    100% |
| Negative |     10% |      0% |

This is a **highly imbalanced distribution**, where one label dominates.

---

### Why Imbalance Causes the Paradox

When one label dominates:

* Both raters tend to choose that label most of the time
* Even **random labeling** (given those tendencies) produces high agreement
* Therefore, **expected agreement (p_e)** becomes very high

---

### Concrete Example (88.9% Agreement)

Consider 9 items:

|                      | Rater B: Label 1 | Rater B: Label 2 | **Total (Rater A)** |
| :------------------- | :--------------: | :--------------: | :-----------------: |
| **Rater A: Label 1** |         0        |         1        |        **1**        |
| **Rater A: Label 2** |         0        |         8        |        **8**        |
| **Total (Rater B)**  |       **0**      |       **9**      |        **9**        |

From this:

* Observed agreement:
  [
  p_o = \frac{8}{9} = 88.9%
  ]

* Marginals:

  * Rater A: 88.9% Label 2
  * Rater B: 100% Label 2

* Expected agreement:
  [
  p_e \approx 88.9%
  ]

Thus:

[
\kappa = \frac{0.889 - 0.889}{1 - 0.889} = 0
]

---

### Intuition

If both raters overwhelmingly choose the same label:

* They will **agree often**
* But not because they are making nuanced distinctions
* Simply because the **label dominates the dataset**

Kappa interprets this as:

> “This agreement was expected anyway.”

---

## 3. Is Your Rubric Reliable?

The answer depends on **what kind of reliability you care about**.

---

### 3.1 Discriminative Power (Analytical Reliability)

If your goal is to evaluate:

* Fine-grained distinctions
* Ability to detect rare cases
* Balanced classification performance

Then:

* **κ = 0 is concerning**
* It suggests:

  * The rubric may not distinguish categories well
  * Or the dataset is too imbalanced to test discrimination

---

### 3.2 Operational Consistency (Practical Reliability)

If your goal is:

* Consistency across raters
* Stability of labeling outcomes
* Real-world deployment behavior

Then:

* **88.9% agreement is strong**
* It indicates:

  * Raters are applying the rubric consistently
  * The system behaves predictably

---

### Key Insight

> High agreement + low κ usually indicates **label imbalance**, not necessarily poor rubric quality.

---

## 4. When Kappa Becomes Misleading

Kappa is sensitive to several conditions:

1. **Extreme class imbalance (prevalence effect)**
2. **Highly skewed marginal distributions**
3. **Small sample sizes**
4. **Systematic label bias (one rater always prefers a label)**

This leads to:

* **Underestimation of agreement quality**
* Apparent contradictions between metrics

This is why the phenomenon is often called:

* **Kappa Paradox**
* **Prevalence Problem**

---

## 5. Defending Raw Agreement Over Kappa

When reviewers question the use of raw agreement, you can provide a **mechanically grounded defense** based on the mathematics of kappa.

---

### Key Argument: The Prevalence Effect

> When one category dominates the marginal distribution, expected agreement ((p_e)) becomes artificially high. This compresses the numerator of kappa ((p_o - p_e)), often driving κ toward zero—even when observed agreement is high.

---

### Example Defense Statement

> **Key Finding Grounding:**
> While the `tone_quality` dimension yields a (\kappa = 0.000), this result is a known statistical artifact of the **Kappa Paradox**, driven by extreme label imbalance. In this dataset, one category dominates the marginal distribution, which inflates the expected chance agreement ((p_e)). As a result, even high observed agreement (88.9%) produces a low kappa value.
>
> Because the primary objective of this rubric is to ensure **operational consistency across labeling rounds**, rather than discriminative performance on a balanced dataset, raw agreement provides a more faithful representation of reliability in this context.

---

## 6. Best Practices

### 6.1 Always Report Multiple Metrics

Do not rely on a single measure:

* Raw agreement
* Cohen’s κ
* Label distribution

---

### 6.2 Show the Confusion Matrix

This reveals:

* Imbalance
* Systematic biases
* Where disagreements occur

---

### 6.3 Consider Alternative Metrics

More robust under imbalance:

* **Gwet’s AC1 / AC2**
* **Krippendorff’s Alpha**

---

### 6.4 Align Metrics with Goals

| Goal                         | Recommended Metric |
| ---------------------------- | ------------------ |
| Consistency                  | Raw agreement      |
| Chance-corrected reliability | Kappa              |
| Imbalanced robustness        | Gwet’s AC1         |

---

## 7. Summary Table: Which Metric to Trust?

| Scenario              | Raw Agreement | Cohen's Kappa | Interpretation                                     |
| :-------------------- | :------------ | :------------ | :------------------------------------------------- |
| **Balanced Labels**   | High          | High          | Strong reliability                                 |
| **Imbalanced Labels** | **High**      | **Low/Zero**  | **Kappa Paradox** – agreement is real but expected |
| **Any Distribution**  | Low           | Low           | Poor reliability                                   |

---

## Conclusion

A kappa value of zero does not automatically imply failure. In many real-world annotation tasks, **label imbalance is expected**, and raters agreeing on the dominant category is both natural and desirable.

The key is not to rely blindly on a single metric, but to interpret:

* Agreement
* Chance expectations
* Label distributions

**together**.

By presenting this context clearly, you can demonstrate that your rubric is functioning correctly—even when kappa appears misleading.

---

## References & Further Reading

1. Cohen, J. (1960). *A Coefficient of Agreement for Nominal Scales.* Educational and Psychological Measurement.
2. Feinstein, A. R., & Cicchetti, D. V. (1990). *High agreement but low kappa: I. The problems of two paradoxes.* Journal of Clinical Epidemiology.
3. Byrt, T., Bishop, J., & Carlin, J. B. (1993). *Bias, prevalence and kappa.* Journal of Clinical Epidemiology.
4. Gwet, K. L. (2014). *Handbook of Inter-Rater Reliability.*
5. Viera, A. J., & Garrett, J. M. (2005). *Understanding interobserver agreement: the kappa statistic.* Family Medicine.
6. Warrens, M. J. (2015). *Cohen’s kappa paradox.*
7. Artstein, R., & Poesio, M. (2008). *Inter-coder agreement for computational linguistics.* Computational Linguistics.
