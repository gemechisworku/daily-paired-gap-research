# Morning Call Summary — Day 1

**Written by:** Nahom
**Confirmed by:** Gemechis Worku
**Date:** 2026-05-07

---

## What each question looked like before the call

**Nahom's draft question (LoRA):**
Originally framed as "why does LoRA work and what rank should I use?" — too broad, could be answered by reading the paper abstract.

**Peer's draft question (DPO):**
Originally framed around "does DPO have reward model overoptimization?" — implied yes/no answer, did not name the specific metric divergence pattern.

---

## What was ambiguous and how each question was sharpened

**On Nahom's question:**
Gemechis pushed back: "you say 'carry signal' — signal for what exactly? Preference-discrimination or general language understanding? And are you asking about all seven modules or specifically the ones relevant to rubric scoring?" The question was narrowed from "why does LoRA work and what rank should I use?" to two decisions in `train_critic.py` L54–58: (1) which of the seven projection modules carry preference-discrimination signal for a rubric-scoring task, and (2) whether r=16 targeting all seven is defensible for 306 pairs or a copy-paste default that could explain the null held-out result.

**On Gemechis's question:**
Nahom pushed back: "are you asking whether overoptimization can happen in DPO at all, or are you asking why it produces the specific mean-up / pass-down pattern you observed?" Gemechis clarified: the metric divergence pattern is the concrete puzzle — the mechanism that explains it is the gap. The question was sharpened from "does DPO have overoptimization?" to three sub-questions: (1) what is DPO's implicit reward and how can it be proxy-hacked, (2) what operational signature in held-out metrics distinguishes soft-dim proxy overoptimization from noise overfit or OOD shift, (3) what concrete dials fix each signature. Artifact pointer tightened to two model card sections that needed rewriting.

---

## Final committed questions

- **Nahom's final question:** see `question.md` in this folder
- **Gemechis's final question:** received via repo — see `pair_DAY_1/question.md` in Gemechis's repo

---

## Attest

Both partners confirm the above summary is accurate and the questions are locked for the day.

[x] Nahom attests
[x] Gemechis Worku attests
