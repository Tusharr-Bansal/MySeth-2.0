# RQ3: Is uncertainty coupled across tasks in a shared encoder?

**Question.** If murmur detection and outcome prediction share an encoder, does the model's confidence on one task predict its reliability on the other? In other words, is uncertainty task-coupled or task-independent?

**Answer, in short.** Murmur and outcome uncertainty are moderately coupled, and recording quality or the diagnosis do not explain it away. However, the coupling comes mostly from the recordings themselves, not from sharing the encoder. One task's confidence is only a weak guide to the other's reliability, so each task needs its own uncertainty score in the confidence gate.

RQ3 also trains the **MySteth 2.0 system model**: one encoder, started from the Domain-SSL weights chosen in RQ2 and fine-tuned jointly on both tasks. RQ4, RQ1, RQ5, RQ6 and RQ7 all use this model.

The full write-up is in the report: [`MySteth2_RQ3_MultiTask_Uncertainty.pdf`](MySteth2_RQ3_MultiTask_Uncertainty.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_RQ3_MultiTask_Uncertainty.pdf`](MySteth2_RQ3_MultiTask_Uncertainty.pdf) | The RQ3 report |
| [`code/rq3-redone.ipynb`](code/rq3-redone.ipynb) | The notebook, as run on Kaggle |
| [`outputs/rq3_outputs/`](outputs/rq3_outputs/) | Result tables (CSV), figures, predictions, temperatures, thresholds and the training log. It also holds copies of the shared folds (`rq2_folds.csv`, identical to [`shared/`](../shared/)) and of the windows file (`rq2_windows.csv`), which the later RQs read from here. |

**Trained networks.** The 9 system-model networks (`joint_domain_f{0,1,2}_s{42,123,7}.pt`) are in Release [models-v1](https://github.com/Tusharr-Bansal/MySeth-2.0/releases/tag/models-v1). The joint-scratch networks and the per-unit output files stay on Kaggle; they are listed with the notebook's inputs in [`large_files/RQ3_Large_Files.md`](../large_files/RQ3_Large_Files.md).

## How it was run

- **Four models, analysed identically:**

  | Model | What it is | Where it comes from |
  |---|---|---|
  | **Joint, Domain-SSL (the system model)** | one encoder from the Domain-SSL weights, fine-tuned jointly on murmur + outcome | trained here |
  | Joint, scratch | the same, without pretraining (RQ6's "− SSL") | trained here |
  | Separate | two per-task Domain-SSL fine-tuned encoders (RQ6's "− multi-task") | reused from RQ2 |
  | Shared frozen | one frozen Domain-SSL encoder with two heads | reused from RQ2 |

- **Training:**
  - Carried over from RQ2: folds, windows, recipe, validation thresholds, patient aggregation (murmur: highest recording; outcome: mean), and 3 folds × 3 seeds.
  - Joint loss: murmur + outcome cross-entropy with equal weights. Unknown-murmur recordings count only toward the outcome term.
  - One best epoch per network, chosen by the summed validation loss, so there is a single network, as deployed.
- **Uncertainty.** The primary measure is the entropy of the temperature-scaled 3-seed ensemble (`ens_temp`). MC-Dropout entropy (40 passes), MC mutual information and ensemble spread are secondary.
- **Analyses:**
  1. **Cross-task correlation:** Spearman ρ between murmur and outcome uncertainty, raw and partial (controlling SQA `p_good` and both true labels), at recording and patient level.
  2. **Does sharing cause it?** Paired differences in coupling between the four models.
  3. **Cross-task error detection:** does one task's uncertainty flag the other task's errors (AUROC)?
  4. **Failure modes:** each task's uncertainty split into thirds, giving a 3 × 3 table compared with independence. The accuracy of the confident task is checked when the other task is uncertain.
  5. **Cost of joint training:** performance of the four models (used by RQ6).
- **Statistics.** 1,000 patient-group bootstrap resamples, paired across models.
- **Kaggle.** Notebook `rq3-redone`, GPU on. Training is saved unit by unit, and the run stops cleanly after about 10.75 h, then resumes. It reads RQ2 part a's output (folds, encoders), RQ2 part b's output (windows, separate and frozen units), CirCor, and the out-of-fold SQA predictions.

## Main results

**Coupling of murmur and outcome uncertainty (Spearman ρ, primary measure):**

| Model | Recording ρ | Recording partial ρ | Patient ρ | Patient partial ρ |
|---|---|---|---|---|
| **System model** | **0.293** | **0.336** | **0.367** | **0.340** |
| Joint, scratch | 0.424 | 0.461 | 0.522 | 0.492 |
| Separate | 0.207 | 0.264 | 0.328 | 0.309 |
| Shared frozen | 0.322 | 0.374 | 0.348 | 0.354 |

- **Coupling is real.** Every model is coupled, and controlling for quality and diagnosis does not remove it.
- **Most of it does not come from sharing.** Separate encoders already reach about 70–90 % of the system's level.
  - Sharing adds a small amount per recording (+0.086 [+0.051, +0.121]) and no clear amount per patient.
  - The system model is no more coupled than a frozen shared encoder.
  - Only joint training without pretraining makes coupling much stronger (+0.217 over separate).
- **One task's confidence is a weak guide to the other.**

  | | AUROC |
  |---|---|
  | Murmur uncertainty → outcome errors | 0.57 |
  | Outcome uncertainty → murmur errors | 0.62 |
  | Each task's own uncertainty → its own errors | murmur 0.82, outcome 0.65 |

  Separate encoders show the same pattern.
- **Discordant cases are rarer than chance.** Confident on one task and uncertain on the other happens in about 6 % of patients each way, about half the rate expected under independence. In those cases, the confident task shows no clear accuracy drop.
- **Joint training has no robust cost compared with separate encoders.** Outcome AUC is unchanged, and murmur F1 and accuracy are slightly higher (+0.016 to +0.032). Compared with joint scratch, the system model is better on murmur F1 and accuracy under every UQ method.
- **Temperatures.** The system model is overconfident on outcome (mean T 1.49) and close to calibrated on murmur (1.02), so temperature scaling matters for outcome; RQ4 takes this up.

## Findings

1. **Uncertainty in MySteth 2.0 is moderately coupled across tasks**, independently of recording quality and diagnosis.
2. **The link lives mostly in the recordings, not in the shared encoder.** Multi-task learning neither creates nor removes a strong cross-task uncertainty link.
3. **Each task needs its own uncertainty score** in the confidence gate. A discordant case is worth noting, but it is not by itself evidence that the confident result is wrong.
4. **Multi-task learning earns its place on efficiency and a small murmur gain.** It uses one encoder and one pass for both tasks, at no robust cost to either. This joint Domain-SSL model is the system model used in the rest of the study.

For the full tables with confidence intervals, the training details, the secondary measures and the limitations, see the [report](MySteth2_RQ3_MultiTask_Uncertainty.pdf).
