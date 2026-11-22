# RQ7: Zero-shot external validation on ZCHSound

**Question.** Does the finished MySteth 2.0 system, trained only on CirCor, still separate children with congenital heart disease (CHD) from healthy children on an independent paediatric dataset, and do its two gates still do their job there?

**Answer, in short.** Yes, the system transfers. The murmur head separates CHD from Normal at **AUC 0.767** over all 1,259 children (**0.827** on the clean set), with almost no false alarms. It detects the lesions that commonly produce a loud murmur: **84.5 %** of clean-set VSD children are called murmur Present.

The full write-up is in the report: [`MySteth2_RQ7_External_Validation.pdf`](MySteth2_RQ7_External_Validation.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_RQ7_External_Validation.pdf`](MySteth2_RQ7_External_Validation.pdf) | The RQ7 report |
| [`ZCHSound_..._Congenital_Heart_Disease.pdf`](ZCHSound_Open-Source_ZJU_Paediatric_Heart_Sound_Database_With_Congenital_Heart_Disease.pdf) | The ZCHSound paper (Jia et al., IEEE TBME 2024) |
| [`code/rq7-external.ipynb`](code/rq7-external.ipynb) | The notebook, as run on Kaggle |
| [`outputs/rq7_outputs/`](outputs/rq7_outputs/) | Result tables (CSV) and figures; the report's Section 6 lists which table each file holds |
| [`outputs/ZCHSound/`](outputs/ZCHSound/) | ZCHSound's two label files (clean and noisy sets) |

The ZCHSound audio, the SQA-encoder embeddings of ZCHSound and the notebook's other inputs are not in this repo; they are listed with links in [`large_files/RQ7_Large_Files.md`](../large_files/RQ7_Large_Files.md).

## How it was run

- **Data.** ZCHSound: 1,259 children, one recording each (about 20 s, one site, 8 kHz), with diagnoses confirmed by echocardiography: Normal, VSD, ASD, PFO, PDA. It has a clean set (941 children, 2 days to 14 years) and a noisy set (318 newborns). There are no murmur labels.
- **Zero-shot.** Nothing was trained or tuned on ZCHSound. Every model, temperature, threshold and gate setting comes from CirCor (RQ1–RQ6). The audio is resampled to 4 kHz.
- **The system, as built:**
  1. ① the final SQA model, with its saved threshold (0.55);
  2. ②③ the RQ3 system model, all 9 networks (3 folds × 3 seeds), combined with `ens_temp` and the 3 folds averaged; thresholds are the mean of the 3 folds' CirCor thresholds (murmur 0.407, outcome 0.413, in [`rq7_thresholds.csv`](outputs/rq7_outputs/rq7_thresholds.csv));
  3. ④ the RQ6 confidence gate, which sends the top 20 % of children by combined Chow risk to a clinician.
  
  ⑤ XAI was not run, because it does not change decisions (RQ6) and was evaluated in RQ5.
- **Target.** CHD (any of VSD, ASD, PFO, PDA) vs Normal. The murmur head is the primary predictor; the outcome head is secondary.
- **Compared:** no gates · SQA gate only · confidence gate only · full system, on the same 1,000 bootstrap resamples of children (seed 42).
- **Metrics,** on automated decisions: AUC, F1 (macro), accuracy, weighted accuracy (CHD × 5), sensitivity and specificity, plus workload and the errors left.
- **Checks before the results.** The diagnosis counts match the ZCHSound paper exactly. The SQA features recomputed on CirCor match the saved ones. All 9 networks reproduce RQ3's saved CirCor outputs.
- **Kaggle.** Notebook `rq7-external`, with GPU and Internet on, and an `HF_TOKEN` secret (for HeAR). Its inputs are ZCHSound, CirCor (for the checks), RQ3's output and the SQA model with its helper files. Of the earlier notebooks it reads only RQ3's output; the rules it applies (`ens_temp`, Chow risk, the 20 % gate) were fixed in RQ4 and RQ6.

## Main results

| | Murmur head | Outcome head |
|---|---|---|
| AUC, no gates, all children | **0.767** [0.739, 0.793] | 0.668 [0.635, 0.700] |
| AUC, no gates, clean set | **0.827** [0.797, 0.855] | 0.738 [0.703, 0.772] |
| Specificity (no gates → full system) | 0.997 → **1.000** | 0.567 → 0.634 |

- **Full-system workload:** 1.6 % re-record, 19.7 % clinician, 78.7 % automated.
- **Conservative operating point.** The murmur head called only 2 of 693 Normal children Present, and none in the full system's automated decisions.
- **By lesion** (exploratory, added after the pre-set analysis):
  - **VSD:** in the clean set, 158 of 187 children (84.5 %) are called Present. VSD + PDA is separated from Normal at **AUC 0.946** [0.922, 0.967].
  - **ASD and PFO:** the ASD murmur is soft and may be absent in infants, and PFO usually gives an unremarkable examination. The head calls 21 of 119 ASD and 1 of 70 PFO children Present, but still ranks ASD + PFO above Normal (AUC 0.690).
- **Gates, under the pre-set rule:**

  | Gate | Murmur head | Outcome head |
  |---|---|---|
  | SQA gate | contributes (small; 20 children re-recorded) | no measurable contribution |
  | Confidence gate | mixed: F1 and weighted accuracy better with it, AUC better without it | harmful: AUC and weighted accuracy better without it |

## Findings

1. **MySteth 2.0 transfers to an independent paediatric dataset** without any retraining or tuning, and keeps CirCor's low false-alarm operating point.
2. **Where a lesion commonly produces a loud murmur (VSD), the system detects it well.**
3. **This dataset cannot separate model failure from population shift** in two places:
   - lesions that are usually soft or silent on auscultation (ASD, PFO);
   - newborns, where age and recording condition coincide.
   
   ZCHSound has no murmur labels, so whether a given child's recording contained an audible murmur is unknown.

For the full tables with confidence intervals, the per-lesion analysis, the gate details and the notes on interpretation, see the [report](MySteth2_RQ7_External_Validation.pdf).
