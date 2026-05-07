# Grounding Commit — Day 3

## Nahom's grounding commit (from LoRA explainer)

**Artifact edited:** `model_card.md` — §Training Configuration (LoRA) + `methodology_rationale.md` — §Adapter Architecture
**Commit hash / PR link:** b7c14d5

**What changed and why:**

`model_card.md` previously had no section defending the LoRA configuration. The new paragraph reads:

> **LoRA configuration:** The adapter uses r=16, alpha=32, targeting all seven projection modules (q, k, v, o, gate, up, down) on Qwen3-1.7B. This is the Unsloth default. In retrospect, this configuration is difficult to defend for a 306-pair preference task: spreading rank budget across seven modules gives each approximately 44 training examples of supervision — insufficient to learn nuanced rubric distinctions beyond surface shortcuts. Attention projections (Q, K, V, O) learn which linguistic patterns to attend to; MLP projections (gate, up, down) learn the decision logic mapping patterns to rubric scores. A more defensible configuration for this dataset size would concentrate rank on v_proj + down_proj at r=32, targeting the value-retrieval and decision-boundary pathways while avoiding gradient diffusion. The null held-out reranking result (Δ A = +0.0025, p=0.40) is consistent with a model that learned shortcut features at validation level but lacked the focused capacity to generalize to near-identical outputs on subtle rubric criteria.

`methodology_rationale.md` §"Why SimPO over DPO and ORPO" previously defended the loss function but was silent on adapter architecture. One paragraph was added:

> **Adapter architecture (post-training note):** The LoRA configuration (r=16, all seven modules) was inherited from Unsloth defaults without ablation. Analysis after training suggests that for a 306-pair dataset, this spreads gradient signal too thin — approximately 44 examples per module, insufficient for nuanced preference boundaries. A targeted configuration (v_proj + down_proj, r=32) would have concentrated capacity on attention value retrieval and MLP decision logic, the two pathways most relevant to a rubric-scoring task. This is a named limitation of the current adapter, not a post-hoc rationalization: the 96.9% validation accuracy reflects shortcut learning that did not transfer to held-out reranking.

**Why this edit matters:** the model card and methodology doc previously said nothing about _why_ the adapter was configured as it was. A reviewer running `model.print_trainable_parameters()` would have no way to assess whether the configuration was appropriate. The edit turns a silent default into a named, honest engineering tradeoff.
