# RQ5: Do explanations align with what the model is confident about?

**Question.** When the model is unsure, do its explanations show it? Or does it give equally confident-looking explanations whether it is sure or not, a known failure of explanation methods?

**Answer, in short.** Yes, under the pre-set rule:
- Both heads show **honest uncertainty**: their Grad-CAM maps spread out as the model becomes less sure.
- Murmur maps point to **systole**, where systolic murmurs are, and this focus weakens as confidence falls.
- The sanity check passed, so the maps come from the trained model.

The full write-up is in the report: [`MySteth2_RQ5_Explanations_Confidence.pdf`](MySteth2_RQ5_Explanations_Confidence.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_RQ5_Explanations_Confidence.pdf`](MySteth2_RQ5_Explanations_Confidence.pdf) | The RQ5 report |
| [`code/rq5-version-3.ipynb`](code/rq5-version-3.ipynb) | The notebook, as run on Kaggle |
| [`outputs/rq5_outputs/`](outputs/rq5_outputs/) | Result tables (CSV) and figures, including the case-study figures; the report's Section 9 lists which table each file holds |

The per-window explanation files (`units_xai/`, about 0.2 GB) stay on Kaggle. They are listed with the notebook's inputs in [`large_files/RQ5_Large_Files.md`](../large_files/RQ5_Large_Files.md).

## How it was run

- **Nothing is trained.** The RQ3 system model (3 folds × 3 seeds) explains every decision, on all 27,335 windows and for both heads. Each recording is explained only by the models of the fold that held it out.
- **Explanations:**
  - **Grad-CAM** at layer3 (0.125 s steps) is the main map, showing the evidence for the decision actually made. The 3 seeds' maps are averaged, as in RQ4's ensemble.
  - **Gradient × input** and **phase-SHAP** are checks. Phase-SHAP gives exact Shapley values of the four heart-sound phases and does not depend on the model's internals.
- **Confidence:** RQ4's `ens_temp`, with **entropy** as the main uncertainty measure (as in RQ1 and RQ3). Chow risk and ensemble spread are secondary measures.
- **Heart-sound phases** (S1, systole, S2, diastole): taken from CirCor's own annotation files, so no new segmenter was needed.
- **Four tests:**
  1. Do maps spread out when the model is unsure?
  2. Do the 3 networks' maps agree less when it is unsure?
  3. Do murmur maps point to systole? This is tested on 496 recordings where a systolic murmur was heard at that site.
  4. Are wrong decisions explained with sharp maps, compared with right ones at the same confidence?
- **Sanity check** (Adebayo et al., 2018): reset the model's last layers to random weights; the maps must change.
- **Case studies:** chosen by a rule set in advance, not hand-picked.
- **Statistics:** the same 1,000 patient-group bootstrap resamples as RQ1–RQ4. Correlations are partial ρ, controlling for recording quality (SQA `p_good`) and the true labels.
- **Answer rule, fixed before the run.** The explanations count as honest if uncertainty goes with more spread-out maps or with less agreement between the networks. The 95 % CI must exclude 0.
- **Kaggle:** notebook `rq5-version-3`, GPU on (about 45–75 min). It reads the RQ3 and RQ4 outputs, CirCor and the out-of-fold SQA predictions.

## Main results

| | Murmur | Outcome |
|---|---|---|
| Sanity check: Grad-CAM vs trained map after reset (median ρ; pass < 0.5) | **−0.069**, passed | not run separately (the murmur head was checked; layer4 is shared by both heads) |
| Uncertainty vs map spread (partial ρ) | **+0.125** [+0.068, +0.182] | **+0.431** [+0.390, +0.471] |
| Verdict | honest uncertainty | honest uncertainty |

- **Systole.** On the 496 target recordings, Grad-CAM puts **1.585×** its chance share on systole. That is **1.674×** for murmurs the model detects, and **0.989×** (chance) for the 64 it misses.
- **Alignment weakens as confidence falls.** The less sure the model, the less its map points to systole, under all three uncertainty measures: entropy −0.138, Chow risk −0.282, ensemble spread −0.215.
- **A model-agnostic method agrees.** Phase-SHAP gives systole **1.405×** its time share, and its systole focus correlates with Grad-CAM's (ρ +0.326).
- **Wrong vs right at the same confidence.**
  - Murmur: wrong decisions have slightly sharper maps (−0.020, a small effect).
  - Outcome: on wrong decisions, the 3 networks' maps agree less (−0.026).
- **Follow-up** (exploratory, added after the results):
  - Split by decision, 3 of 4 groups meet the honest-uncertainty rule. Murmur Present decisions are mixed: the networks agree less when unsure, but the maps get sharper.
  - For detected murmurs, confident maps spread their evidence over most heartbeats, while unsure ones rest on fewer beats. Beat coverage falls from 0.91 to 0.65 between the most and the least confident third.

## Findings

1. **The explanations are honest about uncertainty.** When the model is less sure, its Grad-CAM maps look less sure too, in both heads, and the randomisation check shows they reflect the trained model.
2. **Murmur explanations are clinically meaningful.** Detected murmurs are explained by systole, missed murmurs are not, and the systole focus fades as confidence falls.
3. **A real murmur recurs on every beat, and confident explanations reflect that.** Confident detections draw evidence from most heartbeats; unsure ones lean on one or two. This is the strongest link between explanation and confidence in RQ5.
4. **Grad-CAM at layer3 is the XAI module used in the final system (RQ6).** It explains each automated decision and does not change it.

For the full tables with confidence intervals, the case studies, the follow-up analyses and the notes on interpretation, see the [report](MySteth2_RQ5_Explanations_Confidence.pdf).
