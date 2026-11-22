# RQ6: End-to-end integration and ablation

**Question.** Does putting SQA, SSL, multi-task prediction, UQ and XAI into one pipeline with two decision gates (quality gate, confidence gate) give a system that is more clinically trustworthy than any single component? This is judged not only by accuracy but by how well the system knows when to defer to a clinician.

**Answer, in short.** Partly supported: yes for murmur, not for outcome.
- **Murmur:** at the same patient workload, the full system clearly beats the stripped baseline, and the confidence gate, the SQA gate and SSL each contribute.
- **Outcome:** no component measurably improves outcome prediction, which stays at AUC 0.66–0.70 in every configuration.

The full write-up is in the report: [`MySteth2_RQ6_End_to_End.pdf`](MySteth2_RQ6_End_to_End.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_RQ6_End_to_End.pdf`](MySteth2_RQ6_End_to_End.pdf) | The RQ6 report |
| [`code/rq6-redone.ipynb`](code/rq6-redone.ipynb) | The notebook, as run on Kaggle |
| [`outputs/rq6_outputs/`](outputs/rq6_outputs/) | Result tables (CSV) and figures; the report's Section 9 lists which table each file holds |

`rq6_patient_decisions.csv` (every patient's decision in every configuration) is also copied to [`final_system/patient_level/`](../final_system/patient_level/), whose README explains its columns. RQ6 keeps no large files on Kaggle; its inputs are listed in [`large_files/RQ6_Large_Files.md`](../large_files/RQ6_Large_Files.md).

## How it was run

- **Nothing is trained.** The full system and every ablation are rebuilt on CPU from the saved outputs of earlier RQs:
  - window logits from RQ3 (joint Domain-SSL, joint scratch) and from RQ2 part b (separate encoders);
  - the out-of-fold SQA decisions.
  
  Before any result, checks confirm that the rebuilt probabilities, temperatures and thresholds equal RQ3's and RQ4's, and that SQA coverage equals RQ1's (734 of 942 patients).
- **The full system:**
  1. **SQA gate:** a patient with no passing recording is asked to re-record.
  2. **System model:** the joint Domain-SSL encoder with both heads, combined with `ens_temp`.
  3. **Confidence gate:** one gate per patient. Each task's Chow risk becomes a percentile, and the higher of the two is used. The top 20 % of patients reaching the gate go to a clinician.
  4. **XAI:** Grad-CAM explains the automated decisions.
- **Ablations:**
  - each component removed one at a time: − SQA gate, − SSL, − multi-task, − UQ, − confidence gate, − XAI;
  - − SQA gate at the same total workload, where the confidence gate takes the SQA gate's patients;
  - the **floor**: a stripped MySteth 2.0 with one raw network, no SQA gate, no confidence gate and no XAI.
- **Same workload.** The gate defers exactly 20 % of the patients reaching it, ranked without labels, so the component rows are compared at the same number of deferred patients.
- **Metrics,** on automated decisions at patient level: weighted accuracy, F1, accuracy, AUC, ECE, sensitivity and specificity. Outcome also gets the system cost. Workload, errors left and deferral quality are reported descriptively.
- **Statistics.** The same 1,000 patient-group bootstrap resamples as RQ1–RQ5, with every row paired against the full system.
- **Answer rule, fixed before the run.** A component contributes if removing it makes at least one rule metric worse and none better. The rule metrics are weighted accuracy, F1 and ECE, plus system cost for outcome.
- **Kaggle.** Notebook `rq6-redone`, CPU only. It reads the outputs of RQ3, RQ2 part b and RQ4, and the out-of-fold SQA predictions.

## Main results

**Workload of the full system (942 patients):** 22.1 % re-record, 15.6 % clinician, 62.3 % fully automated.

**Murmur, full system vs floor** (automated decisions):

| | Weighted accuracy | F1 | ECE | Accuracy | Sensitivity | Errors left |
|---|---|---|---|---|---|---|
| Floor | 0.850 | 0.868 | 0.081 | 0.915 | 0.771 | 74 |
| **Full system** | **0.912** | **0.935** | **0.036** | **0.955** | **0.864** | **26** |

**Which components contribute (pre-set rule):**

| Component | Murmur | Outcome |
|---|---|---|
| Confidence gate | contributes (weighted accuracy, F1; 30 fewer errors left) | no measurable contribution |
| ① SQA gate | contributes (ECE 0.036 with it, 0.055 without) | no measurable contribution |
| ② SSL | contributes (F1) | no measurable contribution |
| ③ Multi-task | no measurable contribution (small ECE gain, interval includes 0) | harmful by weighted accuracy only (operating point; AUC unchanged) |
| ④ UQ wrapper | no measurable contribution (small ECE gain, interval includes 0) | no measurable contribution |
| ⑤ XAI | does not change decisions by design (explanations evaluated in RQ5) | same |

- **SQA gate vs giving its patients to a clinician.** At the same total workload, the two designs give automated decisions of the same quality. The SQA design does it with **less than half the clinician time**: 15.6 % vs 37.7 % of patients.
- **Patients whose murmur clinicians could not call (Unknown, 68).** The full system decides only 6 of them automatically. The SQA gate sends 56 to re-record and the confidence gate sends 6 to a clinician. With no gates, all 68 would be decided automatically.
- **Outcome.** AUC stays at 0.656–0.699 in every configuration. This is in the range of published audio-only systems on CirCor (e.g. CardioPHON audio-only AUROC 0.677).

## Findings

1. **Integration, not a single component, delivers trustworthy murmur screening.** Three components each contribute in a different way: the confidence gate to accuracy, the SQA gate to calibration, and SSL to F1. This holds at a fixed, reported workload, and the system leaves 26 instead of 74 errors in its automated decisions.
2. **The two gates complement each other.** The SQA gate replaces most clinician reviews with a cheap re-recording at no loss in quality. Together, the gates keep 62 of the 68 patients that clinicians could not call away from an automated decision.
3. **Outcome is the open problem.** Results plateau across all configurations and match published audio-only systems, which points to information beyond the heart-sound recording.

For the full tables with confidence intervals, the design corrections made after the first run, the literature comparison and the notes on interpretation, see the [report](MySteth2_RQ6_End_to_End.pdf).
