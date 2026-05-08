1/6  
I hit a reliability paradox: 88.9% rater agreement, but Cohen's kappa = 0.000.  
That looked wrong until I unpacked the math.

2/6  
Kappa is not raw agreement.  
It measures agreement beyond chance: `kappa = (p_o - p_e) / (1 - p_e)`.  
If expected chance agreement (`p_e`) is already very high, kappa can collapse.

3/6  
This happens under label imbalance.  
If one label dominates, both raters will often pick it, so agreement stays high even with weak discrimination between categories.

4/6  
So high agreement + low kappa is often a prevalence artifact, not automatic rubric failure.  
The real question is what reliability you need: operational consistency or fine-grained discriminative power.

5/6  
Practical reporting fix: never publish one metric alone.  
Report raw agreement, kappa, and label distribution together, plus the confusion matrix.

6/6  
For imbalanced settings, I also include a robustness metric like Gwet's AC1.  
Bottom line: interpret reliability metrics as a system, not a single score.
