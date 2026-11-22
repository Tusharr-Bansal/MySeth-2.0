# 00_SQA: Component 1, Signal Quality Assessment

**Task.** Decide whether a CirCor recording is **Good** (heart sounds usable) or **Poor**, from audio alone, so that later components can gate on it. In MySteth 2.0, a patient with no Good recording is asked to re-record.

**Result, in short.** The final SQA model reaches **F1-macro 0.824, accuracy 0.837, weighted accuracy 0.850 and AUC 0.929**, out of fold on all 3,163 recordings. It is a fusion of five frozen pretrained audio encoders with 7 hand-crafted features. It improves on the previous SQA model type (hand-crafted LightGBM) on all four metrics.

The full write-up is in the report: [`MySteth2_Component1_SQA_Final.pdf`](MySteth2_Component1_SQA_Final.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_Component1_SQA_Final.pdf`](MySteth2_Component1_SQA_Final.pdf) | The SQA report: labelling, validation and results |
| [`code/sqa-model-redone.ipynb`](code/sqa-model-redone.ipynb) | The notebook, as run on Kaggle |
| [`inputs/circor_main_labels.csv`](inputs/circor_main_labels.csv) | The Good / Poor quality label of all 3,163 recordings |
| [`inputs/recording_features_full_v2.csv`](inputs/recording_features_full_v2.csv) | The 7 hand-crafted features per recording |
| [`outputs/sqa_results.csv`](outputs/sqa_results.csv) | Out-of-fold performance of every model, with 95 % CIs |
| [`outputs/sqa_comparisons.csv`](outputs/sqa_comparisons.csv) | Paired differences: fusion minus each other model |
| [`outputs/sqa_oof_predictions.csv`](outputs/sqa_oof_predictions.csv) | **Per recording: out-of-fold `p_good`, its fold threshold and the gate decision.** This is the file RQ1, RQ3, RQ5 and RQ6 read. |
| [`outputs/sqa_final_model.joblib`](outputs/sqa_final_model.joblib) | The final model, refitted on all labels, with threshold 0.55, for scoring new recordings (used in RQ7). Also in Release [models-v1](https://github.com/Tusharr-Bansal/MySeth-2.0/releases/tag/models-v1). |

The encoder embeddings and the other Kaggle inputs are listed with links in [`large_files/00_SQA_Large_Files.md`](../large_files/00_SQA_Large_Files.md).

## How it was run

- **Labels.** The first SQA version used Snorkel weak labels. An independent check showed they agreed with human listening at only κ = 0.423, below the κ ≥ 0.60 reliability bar, so the labels were rebuilt:
  - All 3,163 recordings now carry a gold label: either labelled by ear, or accepted from the weak labeller after a blind spot-check.
  - Every labelling batch passed a blind intra-rater re-check (κ 0.613–0.795).
  - Final set: 2,154 Good and 1,009 Poor.
- **Model.** Five frozen pretrained audio encoders: PANNs embedding, PANNs AudioSet class scores, AST, CLAP and HeAR.
  1. Each recording is cut into windows (10 s; 2 s for HeAR), and each window is embedded by the encoder.
  2. The window embeddings are pooled per recording (mean ‖ max) and joined with the 7 hand-crafted features.
  3. A logistic regression on each encoder's vector gives P(Good).
  4. The five probabilities are averaged (late fusion).
- **Baselines:**
  - hand-crafted logistic regression;
  - hand-crafted LightGBM, the previous SQA model type;
  - each encoder on its own.
- **Leak-free evaluation.** Nested, patient-grouped cross-validation (5 outer × 4 inner folds). Children who appear under two patient IDs are kept in one group (942 patients → 872 groups). The regularisation strength and the Good/Poor threshold are chosen inside the training folds only. Fold thresholds: 0.50–0.625.
- **Metrics.** F1 (macro), accuracy, weighted (balanced) accuracy and AUC, with patient-level bootstrap 95 % CIs. Paired bootstrap is used for the differences between models.
- **Outputs for later components.**
  - Later components read the **out-of-fold** `p_good`, so no recording is scored by a model that saw its own label.
  - A final model refitted on all labels (threshold 0.55, the median fold threshold) is saved for new recordings.
- **Kaggle.** Notebook `sqa-model-redone`, with GPU and Internet on, and an `HF_TOKEN` secret (for HeAR). It reads CirCor, the labels, the hand-crafted features and, optionally, cached embeddings.

## Main results

**Out of fold, all 3,163 recordings:**

| Model | F1-macro | Accuracy | Weighted acc. | AUC |
|---|---|---|---|---|
| Hand-crafted LogReg | 0.813 | 0.827 | 0.838 | 0.900 |
| Hand-crafted LightGBM (previous model type) | 0.800 | 0.812 | 0.832 | 0.907 |
| Best single encoder (CLAP + hc) | 0.816 | 0.830 | 0.839 | 0.919 |
| **SQA fusion (final)** | **0.824** [0.811, 0.838] | **0.837** [0.824, 0.850] | **0.850** [0.838, 0.863] | **0.929** [0.920, 0.939] |

- **Against the previous model type (LightGBM),** fusion is better on all four metrics, and every CI excludes 0 (e.g. AUC +0.022, weighted accuracy +0.019).
- **Against hand-crafted logistic regression,** the AUC gain is clear (+0.029), and the thresholded metrics are within noise. The encoders mainly improve how recordings are ranked, which matters because later components use the continuous `p_good`.
- **Against every single encoder,** fusion is better on AUC.
- **The gate in practice:**
  - It passes 1,864 recordings (58.9 %).
  - It catches 896 of the 1,009 Poor recordings (89 %), and 6.1 % of passed recordings are Poor.
  - At patient level, 22.1 % of patients have no passing recording and are asked to re-record.

## Findings

1. **A reliable label set was the foundation.** Replacing the weak labels with checked gold labels makes the SQA benchmark a measure against what a listener hears.
2. **Frozen pretrained audio encoders, fused with simple features, give a strong quality gate** (AUC 0.929), with a leak-free evaluation and every tuned value chosen inside the training folds.
3. **The gate pays off downstream.** It is selective, but in the integrated system (RQ6) it improves murmur calibration (ECE 0.036 with the gate, 0.055 without). Asking patients to re-record gives automated decisions of the same quality as sending them to a clinician, with less than half the clinician time (15.6 % vs 37.7 % of patients). RQ1 shows how its scores relate to the model's uncertainty.

For the labelling procedure and its reliability checks, the full benchmark and paired comparisons, the final model's details and the limitations, see the [report](MySteth2_Component1_SQA_Final.pdf).
