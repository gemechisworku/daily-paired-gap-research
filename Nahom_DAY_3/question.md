## In DPO (no explicit reward model), what plays the role of "reward model overoptimization," and how would it explain my held-out **mean score going up while pass rate goes down**?

My `ablations/ablation_results.json` shows the trained variant with `mean_score_pct: 97.92` but `pass_rate: 0.82`, against baseline `mean_score_pct: 93.44` / `pass_rate: 0.86` and prompt-only `100.0` / `1.0`. So Delta A is a real mean-score lift (`+4.48`, `95% CI [3.68, 5.44]`, `p≈0.0002`), Delta B is negative (`-2.08`), **and pass rate moved the wrong way** even where mean score moved up. `model_card.md` §Bias, Risks, and Limitations names the residual failure clusters (capacity over-commitment, TCV quoting, discount/promo language) but not a mechanism.

The diagnostic gap is: in classic RLHF, "reward model overoptimization" means the policy hill-climbs a *learned* reward proxy that diverges from the true objective, and **KL regularisation to a reference policy is the standard brake**. In DPO there is **no separate reward model** — there is an *implicit* reward defined by the chosen/rejected log-prob ratio, and `beta` plays the KL-anchor role. So:

1. **What is the analogue of "reward hacking"** when the reward is implicit and tied to the data itself? Concretely, when `rewards/margins` keeps climbing during training, what is the policy actually optimizing that the *held-out evaluator* does not score?
2. **What is the operational signature** in held-out metrics — specifically the pattern *mean score up, pass rate down on a hard-policy slice* — and how is it distinguishable from preference-noise overfit, distribution shift, or just LoRA-rank-too-high?
3. **What concrete dial** (`beta`, fewer epochs, KL-aware variants, reference-policy refresh) is the right corrective for each signature?

Knowing this would let me rewrite the **"Summary" + "Bias, Risks, and Limitations" sections of `model_card.md`** and the **"Delta B" honest-result paragraph in `memo.md`** with a real mechanistic explanation instead of "remaining failures are concentrated in hard-policy honesty slices."

Generalizable because *every* DPO shop will eventually ship a model where aggregate metrics look good and a slice silently regresses, and the mental model "DPO has no reward model so overoptimization does not apply" is a common and wrong intuition.
