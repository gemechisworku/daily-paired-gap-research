# Evening Call Summary - Day 4

Mamaru: Your explainer works best where you show the mechanical bridge from high observed agreement to kappa near zero through high expected agreement under skewed marginals.
Gemechis: I revised it to make the action policy explicit: report raw agreement, kappa, marginals, and confusion matrix together, and add AC1/AC2 when imbalance is strong.
Gemechis: On your possible answer to my pass^1 vs pass^k question, I need a hard split between deterministic misses and stochastic near-misses before any reliability claim.
Mamaru: Agreed; I tightened it so pass^1 is the deployment-truth metric for one-shot systems, while pass^k is a capability-ceiling metric only when the product actually samples multiple candidates and selects among them.
Gemechis: Good, that revision makes both the evaluation claim and the deployment claim consistent.
