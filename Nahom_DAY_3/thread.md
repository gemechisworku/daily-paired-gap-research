1/6  
I fine-tuned a LoRA preference critic for B2B sales emails and hit a weird result: validation looked strong, but held-out reranking was weak.  
The key lesson: good validation can hide shortcut learning when data is small.

2/6  
LoRA is not just a "smaller finetune."  
It learns low-rank updates (`W' = W + BA`), so rank controls how many independent update directions the model can express.  
If rank/capacity is misallocated, generalization breaks first.

3/6  
Defaulting to all 7 transformer projections can spread signal too thin on limited preference pairs.  
With ~306 pairs, the model can memorize easy surface cues instead of learning robust rubric logic.

4/6  
What worked conceptually: treat modules by function.  
Attention-side modules help pattern retrieval ("what to notice"), while MLP-side modules shape decision boundaries ("how to score it").  
So capacity should be concentrated, not sprayed.

5/6  
Practical fix path:  
- reduce target modules  
- increase rank where it matters  
- add LoRA dropout for regularization  
- evaluate with held-out reranking metrics (not only val accuracy)

6/6  
My takeaway: configuration is not just picking `r=16` because it is default.  
For preference critics, the real question is whether adapter capacity matches rubric complexity and data coverage.  
If not, you get "looks good in validation, fails in deployment slices."
