# RQ4: Does confidence-based rejection improve calibration?

**Question.** Does rejecting low-confidence cases genuinely improve calibration on the cases that are kept? Or does it just remove difficult-but-correct predictions and make the system look more reliable than it is?

**Answer, in short.**
- **Murmur:** rejection genuinely improves calibration, well beyond what removing the same number of patients at random gives. It removes errors far faster than correct predictions, and it sends the clinically hard cases (soft grade-I murmurs) to the clinician.
- **Outcome:** rejection improves accuracy and ranking, but not calibration, which is already good before rejection.

The full write-up is in the report: [`MySteth2_RQ4_Rejection_Calibration.pdf`](MySteth2_RQ4_Rejection_Calibration.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_RQ4_Rejection_Calibration.pdf`](MySteth2_RQ4_Rejection_Calibration.pdf) | The RQ4 report |
| [`code/rq4-redone.ipynb`](code/rq4-redone.ipynb) | The notebook, as run on Kaggle |
| [`outputs/rq4_outputs/`](outputs/rq4_outputs/) | Result tables (CSV) and figures; the report's Section 9 lists which table each file holds |

The temperatures and thresholds used by the final system are also copied to [`final_system/model_config/`](../final_system/model_config/). The regenerated MC-Dropout passes (`units_mc/`) stay on Kaggle; they are listed with the notebook's inputs in [`large_files/RQ4_Large_Files.md`](../large_files/RQ4_Large_Files.md).

## How it was run

- **Model.** The RQ3 system model (joint Domain-SSL encoder with both heads, 3 folds × 3 seeds), used unchanged. RQ3's folds, out-of-fold predictions for all 942 patients, validation thresholds and temperatures are carried over.
- **No retraining.** RQ3 had saved MC-Dropout passes only for its seed-42 models. The six seed-123 and seed-7 models were therefore re-run over all windows to draw 40 passes each, and checked against RQ3's saved outputs.
- **Seven UQ methods:**
  - `single`, `temp` and `mc` (40 passes);
  - `mc_temp`, `ens` (3 seeds) and `ens_mc_temp`;
  - **`ens_temp`** (temperature-scaled 3-seed ensemble), the primary method, fixed in advance.
- **Rejection scores.** The primary score is the threshold-aware **Chow** score: p / t below the decision threshold t, and (1 − p) / (1 − t) above it. It was chosen because the decisions use thresholds away from 0.5. Entropy, MC mutual information and ensemble spread are the alternatives.
- **The key control.** Every ECE on the kept patients is compared with **randomly rejecting the same number of patients**, on the same resamples. This way a smaller set alone cannot look like a gain.
- **What is reported:**
  - accuracy–rejection curves from 0 to 90 %, with metrics at 10, 20, 30 and 50 % rejection;
  - ECE before and after rejection, against random rejection;
  - what share of the rejected cases were correct;
  - who is deferred, by class and murmur grade, and the Unknown-murmur patients;
  - how well each score separates wrong from right decisions (AUROC);
  - comparison with De Simone et al. and Zhang et al.
- **Level.** Patient level is the main analysis, recording level the secondary one. Murmur and outcome are gated separately.
- **Statistics.** The same 1,000 patient-group bootstrap resamples as RQ3.
- **Answer rule, fixed before the run.** A gain is genuine if the kept-set ECE is lower than both the ECE before rejection and the ECE after random rejection, with the 95 % CIs excluding 0.
- **Kaggle.** Notebook `rq4-redone`, GPU on (for the MC passes). It reads RQ3's output and CirCor.

## Main results

**Murmur, `ens_temp`, Chow, patient level:**

| Rejection | 0 % | 10 % | 20 % | 30 % | 50 % |
|---|---|---|---|---|---|
| ECE of kept patients | 0.065 | 0.054 | 0.048 | 0.036 | 0.013 |
| ECE after random rejection | – | 0.065 | 0.066 | 0.066 | 0.069 |
| Verdict | | genuine gain | genuine gain | genuine gain | genuine gain |
| Accuracy | 0.911 | 0.943 | 0.960 | 0.964 | 0.968 |
| Sensitivity | 0.799 | 0.818 | 0.859 | 0.881 | 0.893 |

- **Errors, not correct cases, are removed.** At 20 % rejection the model defers 64 % of its murmur errors while losing 16 % of its correct predictions. 71 % of the deferred patients were correct, against 91 % if rejection were random.
- **The hard cases are deferred.** Soft grade-I murmurs, the ones the model gets wrong most often, are deferred at about twice the random rate. Loud grade-III murmurs are almost never deferred (0 of 46 at 20 %).
- **All seven UQ methods** show the murmur calibration gain from 20 % rejection on, and so does entropy as the score.
- **Outcome.** The temperature-scaled outcome model starts well calibrated (ECE 0.049), and up to 30 % rejection no method differs from random rejection in calibration. Rejection still raises accuracy (0.575 → 0.616 at 30 %) and AUC at every rate (0.652 → 0.678 at 20 %).
- **Score choice.** Chow ranks errors best where the threshold is far from 0.5: murmur per recording, AUROC 0.839 vs 0.807 for entropy. Where thresholds are closer to 0.5 (per patient, and outcome), entropy ranks errors better. The verdicts are the same under both scores.
- **Literature.** At about 20 % rejection, our murmur accuracy rises by +4.9 points and sensitivity by +6.0. De Simone et al. report +0.22 and +3.03, rejecting 2-s segments rather than patients, so this is a reference point.

## Findings

1. **For murmur, the confidence gate delivers real reliability, not just apparent reliability.** Calibration improves beyond random rejection at every rate, and errors are removed several times faster than correct predictions.
2. **What it defers is what a clinician should see.** Soft murmurs that the model often misses go to the clinician, while clear loud murmurs stay automated. Cases that clinicians themselves could not call (Unknown) are also deferred more than the rejection rate, clearly so from 30 % per patient and at every rate per recording.
3. **For outcome, the gate should be judged on accuracy, sensitivity and deferral, not on calibration gains.** Calibration is already good before rejection.
4. **This gate became MySteth 2.0's second gate.** `ens_temp` with the Chow score is the confidence gate used in RQ6 and RQ7.

For the full tables with confidence intervals, the sensitivity checks, the reliability diagrams, the literature comparison and the limitations, see the [report](MySteth2_RQ4_Rejection_Calibration.pdf).
