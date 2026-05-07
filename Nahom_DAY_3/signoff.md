# Sign-off — Day 3

## Nahom's sign-off on Gemechis's LoRA explainer

**Asker:** Nahom
**Gap closure judgment:** closed

**What I understand now that I did not before:**

Before this explainer I knew I had chosen r=16 across all seven projection modules because it was the Unsloth default. I could not explain what each module contributed to the critic's task or why that rank was right or wrong for 306 pairs.

I now understand: attention projections (Q, K, V, O) learn _which patterns to attend to_ — surface features like whether the email opens with a value proposition or uses generic language. MLP projections (up, gate, down) learn _how to combine those patterns into a decision_ — the "feature X implies decision Y" logic that maps to rubric criteria like "specific credible ROI claim → pass." For a rubric critic, both are needed but for different reasons.

On rank: r=16 gives the adapter 16 independent directions of adaptation per weight matrix. That is not inherently too low, but spread across 7 modules with only 306 training pairs, each module receives roughly 44 examples of supervision — too few to learn nuanced rubric distinctions. The 96.9% validation accuracy was achievable because shortcut features (percentage presence, email length, certain phrases) are learnable with 44 examples. Reranking requires distinguishing near-identical outputs on subtle criteria — that requires more focused capacity.

The defensible fix for a 306-pair task is: target fewer modules (v_proj + down_proj concentrates attention value retrieval and MLP decision logic) at higher rank (r=32), not all seven at r=16. The Unsloth default is only appropriate with >1,000 pairs where each module receives adequate supervision.

This changes the `model_card.md` LoRA-configuration paragraph from "defaults were used for reproducibility" to a defended choice with named tradeoffs.
