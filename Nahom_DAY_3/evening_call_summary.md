# Evening Call Summary — Day 1

**Written by:** Nahom
**Confirmed by:** Gemechis Worku
**Date:** 2026-05-07

---

## Feedback on Nahom's explainer (DPO overoptimization)

*Nahom wrote the explainer for Gemechis's DPO question. Gemechis is the asker giving feedback here.*

**What landed:**
The diagnostic table (metric pattern → most likely cause) landed immediately — Gemechis said it gave him a framework to reason about his results rather than just describe them. The implicit reward formula `r*(x,y) = β·log(π_θ/π_ref)` made the "DPO has no reward model" intuition concretely wrong: seeing the formula made clear the policy ratio IS the reward model. The two ready-to-paste model card paragraphs were the most directly useful part — Gemechis could drop them into §Summary and §Limitations with minor edits.

**What did not land / what was unclear:**
The IPO section was too compressed. "Squared loss saturates" did not carry the intuition on its own — Gemechis did not understand why saturation prevents overoptimization without knowing that DPO's logistic loss has no ceiling on margin growth. The connection between the `rewards/margins` curve rising during training and the held-out metric divergence needed one bridging sentence explaining that as training progresses, KL divergence accumulates and eventually crosses the threshold where proxy and true objective decouple.

**What Nahom revised in response:**
Added one sentence before the IPO paragraph: "DPO's logistic loss can be driven to zero by pushing the chosen-vs-rejected margin to infinity — there is no ceiling, so the optimizer never stops." Then expanded the IPO sentence to: "IPO replaces this with a squared loss whose gradient shrinks as margin grows, so the optimizer self-limits before the margin exceeds what the data supports." Also added one bridging sentence in the Gao section: "In practice this means: as training epochs accumulate, KL from π_ref grows, and past a threshold the continuous rubric score keeps rising while pass_rate — which includes hard-fail criteria the preference pairs undersupplied — begins to fall."

---

## Feedback on Gemechis's explainer (LoRA mechanics)

*Gemechis wrote the explainer for Nahom's LoRA question. Nahom is the asker giving feedback here.*

**What landed:**
The attention vs. MLP split was the clearest framing Nahom had seen — "attention projections learn what patterns to look for; MLP projections learn how to combine patterns into decisions" maps directly onto the rubric structure (surface features vs. criterion relationships). The gradient diffusion hypothesis (306 pairs ÷ 7 modules ≈ 44 examples per module) made the configuration problem concrete and measurable. The diagnostic probe code (LoRA weight norms per module) is immediately runnable against the shipped `training/adapter_seed11/`.

**What did not land / what was unclear:**
The rule-of-thumb formula (`r × num_modules × num_layers ≈ 2 × criteria × pairs / 10`) was hard to follow without a worked example anchored to Nahom's actual numbers (32 layers, 7 modules, 10 rubric dimensions, 306 pairs). The conclusion (r ≈ 5–10 or r=16–32 with fewer modules) felt like a wide range without a clear recommendation for this specific case.

**What Gemechis revised in response:**
Added a worked example pinned to Nahom's actual numbers immediately after the rule-of-thumb formula: "For Nahom's setup — 32 layers, 7 modules, 10 rubric dimensions, 306 pairs — the formula gives r × 7 × 32 ≈ 612, so r ≈ 2–3. That is unrealistically low because the formula assumes perfect signal; a practical floor of r=8 gives r=8 targeting 3 modules (v_proj + down_proj + o_proj) as the minimum defensible config. r=16 across all 7 is 3.5× over-parameterized relative to the dataset size." Also replaced the wide range conclusion ("r ≈ 5–10 or r=16–32 with fewer modules") with a direct recommendation: "For 306 pairs, the defensible choice is r=16 targeting v_proj + down_proj only, or r=32 if you want headroom. Do not spread rank across all seven modules without at least 1,000 pairs."

---

## Post-revision state

- Nahom's explainer (DPO): revised — IPO paragraph expanded, one bridging sentence added in Gao section
- Gemechis's explainer (LoRA): revised — worked example added, rank recommendation tightened to a direct call
