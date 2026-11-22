# RQ2: Does self-supervised pretraining improve calibration?

**Question.** Does self-supervised pretraining (BYOL-A) give paediatric heart-sound models that are better calibrated, not just more accurate? Following the methods document, RQ2 also tests domain-shift pretraining: adding adult PhysioNet 2016 and PASCAL recordings to CirCor.

**Answer, in short.**
- **Accuracy:** slightly better, but only with in-domain data. CirCor-SSL, fine-tuned, gives a small, clear murmur gain over training from scratch.
- **Calibration:** no. No pretrained encoder is better calibrated than Scratch once standard post-hoc calibration is applied.
- **Domain shift:** no. The adult data does not help, and as frozen features it hurts.
- **What drives calibration** is the training regime (frozen vs fine-tuned), not pretraining. Calibration therefore has to come from the UQ component (RQ4).

RQ2 also decides, by a rule fixed in advance, that the MySteth 2.0 system model (trained in RQ3) **starts from the Domain-SSL encoder**.

The full write-up is in the report: [`MySteth2_RQ2_SSL_vs_Scratch.pdf`](MySteth2_RQ2_SSL_vs_Scratch.pdf).

## What is in this folder

RQ2 has two notebooks: **part a** pretrains the encoders and **part b** trains and compares the encoder arms.

| Path | Contents |
|---|---|
| [`MySteth2_RQ2_SSL_vs_Scratch.pdf`](MySteth2_RQ2_SSL_vs_Scratch.pdf) | The RQ2 report (covers both parts) |
| [`code/rq2-part-1-again.ipynb`](code/rq2-part-1-again.ipynb) | **Part a:** builds the shared folds file and pretrains the six BYOL-A encoders |
| [`code/rq2-part-2-again.ipynb`](code/rq2-part-2-again.ipynb) | **Part b:** trains and evaluates the five encoder arms on both tasks, and applies the system rule |
| [`outputs/part_a/rq2a_outputs/`](outputs/part_a/rq2a_outputs/) | Part a: the folds file (`rq2_folds.csv`), corpus summaries, pretraining loss and its figure |
| [`outputs/part_b/rq2_outputs/`](outputs/part_b/rq2_outputs/) | Part b: results, paired differences, predictions, temperatures, thresholds, training log, the windows file, the system choice (`rq2_system_choice.json`) and figures |

`rq2_folds.csv` is the folds file used by every model from RQ2 onward. The copies in part a, part b and [`shared/`](../shared/) are identical.

The six pretrained encoders, the 90 trained networks and the per-unit output files stay on Kaggle. They are listed with both notebooks' inputs in [`large_files/RQ2_Large_Files.md`](../large_files/RQ2_Large_Files.md).

## How it was run

**Part a: pretraining** (`rq2-part-1-again`)
- **Folds.** 3 folds split by patient group, with repeat-visit children kept together (942 patients → 872 groups), and balanced on murmur × outcome. Inside each fold: 75 % train, 10 % validation, 15 % calibration. Saved once as `rq2_folds.csv`.
- **BYOL-A,** one encoder per fold and corpus, never seeing that fold's test recordings:
  - **CirCor-SSL:** the fold's non-test CirCor recordings (about 2,100);
  - **Domain-SSL:** the same, plus PhysioNet 2016 (3,240) and PASCAL (828), about 6,170 in total.
- **Matched budget.** 7,500 updates for every encoder, so the two corpora differ only in data, not in the amount of training. All six encoders trained smoothly, and none collapsed.
- **Kaggle.** GPU on; about 2.8 GPU hours per encoder. The run stops cleanly after about 10.75 h and resumes from checkpoints. Inputs: CirCor, PASCAL and PhysioNet 2016.

**Part b: the five arms** (`rq2-part-2-again`)
- **Arms.** All five use the same network, recipe, budget and uncertainty pipeline:

  | Arm | Starting weights | What is trained |
  |---|---|---|
  | Scratch | random | all layers |
  | CirCor-SSL frozen | CirCor-SSL | head only |
  | CirCor-SSL fine-tuned | CirCor-SSL | all layers |
  | Domain-SSL frozen | Domain-SSL | head only |
  | Domain-SSL fine-tuned | Domain-SSL | all layers |

- **Training.**
  - Each arm is trained for murmur (Present/Absent) and outcome (Normal/Abnormal), 3 folds × 3 seeds.
  - Unweighted loss; up to 60 epochs, keeping the best validation epoch, with patience 15.
  - The decision threshold is chosen on validation patients.
  - Unknown-murmur patients are predicted but not trained on.
- **Prediction.** The whole recording is cut into 5 s windows every 2.5 s, and the window probabilities are averaged. Recordings are combined per patient: murmur takes the highest recording, outcome the mean.
- **Uncertainty (RQ4's definitions):** `single`, `temp`, `mc` (40 passes), `ens` (3 seeds), `ens_temp`.
- **Metrics:**
  - patient level: F1 (macro), accuracy, weighted accuracy, AUC and outcome cost;
  - recording level: ECE, Brier and log-loss.
- **Statistics.** 1,000 patient-group bootstrap resamples, the same for every arm, so differences between arms are paired.
- **System rule, fixed before the run.** Start RQ3's shared encoder from Domain-SSL, unless CirCor-SSL fine-tuned is clearly better on murmur weighted accuracy.
- **Kaggle.** GPU on. Inputs: CirCor and part a's output (folds and the six encoders). The Scratch arm is reused unchanged from the earlier run `rq2-redone`, after checking that its folds file is identical.

## Main results

**Murmur, patient level, `ens_temp`:**

| Arm | Weighted acc. | F1 | Accuracy | AUC |
|---|---|---|---|---|
| Scratch | 0.862 | 0.842 | 0.890 | 0.928 |
| CirCor-SSL frozen | 0.849 | 0.834 | 0.886 | 0.906 |
| **CirCor-SSL fine-tuned** | **0.879** | **0.875** | **0.916** | 0.930 |
| Domain-SSL frozen | 0.790 | 0.737 | 0.796 | 0.885 |
| Domain-SSL fine-tuned | 0.868 | 0.841 | 0.888 | 0.931 |

- **Pretraining.** CirCor-SSL fine-tuned beats Scratch on murmur F1 (+0.033 [+0.007, +0.059]) and accuracy (+0.026). Weighted accuracy and AUC show no clear difference. Domain-SSL fine-tuned is indistinguishable from Scratch.
- **Domain shift.** Frozen Domain-SSL features are clearly worse than frozen CirCor-SSL for murmur (weighted accuracy −0.059, F1 −0.098). Fine-tuning recovers them to Scratch level, but not to CirCor-SSL fine-tuned.
- **Calibration.**
  - Murmur: no SSL arm is better calibrated than Scratch, which has the lowest ECE (0.020).
  - Outcome: raw probabilities from frozen SSL encoders are better than Scratch's, but after temperature scaling or ensembling every calibration difference disappears.
- **Temperatures.** Frozen encoders are slightly underconfident (mean T 0.84–0.98). Fine-tuned encoders are overconfident on both tasks (1.37–1.76), whichever corpus they started from.
- **Outcome is hard for every arm** (patient AUC 0.65–0.67). Domain-SSL fine-tuned is clearly better than CirCor-SSL fine-tuned on outcome cost (−634).
- **System decision.** CirCor-SSL fine-tuned is not clearly better on murmur weighted accuracy (+0.011 [−0.013, +0.035]), so RQ3 starts from the **Domain-SSL** encoders. On the full evidence each corpus wins one task: CirCor-SSL on murmur F1 and accuracy, Domain-SSL on outcome.

## Findings

1. **Small-corpus SSL mainly gives a better starting point for fine-tuning, not better uncertainty.** The accuracy gain is small and in-domain only.
2. **Pretraining does not make the models better calibrated** once post-hoc calibration is applied. Calibration has to be secured by the UQ component (RQ4), not expected from the representation.
3. **The training regime drives calibration:** frozen encoders are underconfident and fine-tuned ones overconfident. Post-hoc calibration is therefore essential for any fine-tuned MySteth 2.0 model.
4. **Domain shift does not help paediatric murmur detection.** With the budget matched, adult and external recordings make frozen features worse.
5. **The system model starts from Domain-SSL,** chosen by a rule fixed before the run. In RQ6, removing SSL from the integrated system clearly lowers murmur F1.

For the full tables with confidence intervals, the calibration differences, reliability diagrams, training diagnostics and limitations, see the [report](MySteth2_RQ2_SSL_vs_Scratch.pdf).
