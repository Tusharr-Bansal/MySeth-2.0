# RQ1: Does SQA filtering change what the model is uncertain about?

**Question.** When a recording passes the SQA gate but its quality is only borderline, does the downstream model's confidence reflect that remaining quality risk? Or are SQA and uncertainty estimation blind to each other?

**Answer, in short.**
- **Murmur:** yes, and honestly. Recordings that only just pass the gate get higher uncertainty, and the model is not falsely confident on them.
- **Outcome:** among accepted recordings, quality does not change outcome accuracy, so there is nothing for the confidence to reflect.
- **SQA and uncertainty are related but mostly flag different recordings.** This supports keeping both gates (RQ6).

The full write-up is in the report: [`MySteth2_RQ1_SQA_Uncertainty.pdf`](MySteth2_RQ1_SQA_Uncertainty.pdf).

## What is in this folder

| Path | Contents |
|---|---|
| [`MySteth2_RQ1_SQA_Uncertainty.pdf`](MySteth2_RQ1_SQA_Uncertainty.pdf) | The RQ1 report |
| [`code/rq1-version-3.ipynb`](code/rq1-version-3.ipynb) | The notebook, as run on Kaggle |
| [`outputs/rq1_outputs/`](outputs/rq1_outputs/) | Result tables (CSV) and figures |

RQ1 keeps no large files on Kaggle; its inputs are listed in [`large_files/RQ1_Large_Files.md`](../large_files/RQ1_Large_Files.md).

## How it was run

- **Nothing is trained.** RQ1 joins two sets of saved scores, recording by recording:
  - the final SQA model's out-of-fold quality score (`p_good`), its fold threshold and its gate decision (from [`00_SQA`](../00_SQA/));
  - the uncertainty of the MySteth 2.0 system model, from RQ4.
- **Uncertainty.** The primary measure is the entropy of the temperature-scaled 3-seed ensemble (`ens_temp`). Chow risk, MC mutual information and ensemble spread are secondary. Murmur and outcome are analysed separately.
- **Borderline vs high.** Passed recordings are ranked by how far they sit above their own fold's SQA threshold. The lowest third is *borderline*, the top third *high*; a median split is used as a sensitivity check.
- **"Falsely confident"** is measured with the overconfidence gap (mean confidence minus accuracy), not with accuracy alone.
- **Ground-truth check.** 113 recordings that humans labelled Poor were passed by the gate. They are compared with the passed recordings labelled Good.
- **Gate on vs off,** at patient level. With the gate on, each patient's prediction is rebuilt from passed recordings only, and patients with none are asked to re-record. This is compared with:
  - the gate off;
  - the gate off on the same patients;
  - random rejection of the same share of recordings;
  - the confidence gate at the same coverage.
- **Statistics.** Spearman ρ, raw and partial (controlling the murmur and outcome labels), with the same 1,000 patient-group bootstrap resamples as RQ3 and RQ4.
- **Answer rule, fixed before the run.**
  - *Reflects the quality risk*: borderline entropy is above high entropy, and the partial ρ of `p_good` with entropy among passed recordings is below 0, with 95 % CIs excluding 0.
  - *Falsely confident*: the borderline overconfidence gap is above the high one.
- **Kaggle.** Notebook `rq1-version-3`, no GPU. It reads the out-of-fold SQA predictions and RQ4's output.

## Main results

**The SQA gate on CirCor.** It catches 896 of the 1,009 human-Poor recordings (89 %). Only 6.1 % of passed recordings are human-Poor.

**Murmur, passed recordings:**

| | Borderline | High | Borderline − high |
|---|---|---|---|
| Mean entropy (bits) | 0.365 | 0.317 | **+0.048** [+0.009, +0.087] |
| Accuracy | 0.861 | 0.890 | −0.030 (not clear) |
| Overconfidence gap | −0.007 | −0.010 | +0.003 (no change) |

- **The uncertainty rises in step with the accuracy drop,** so the model is not falsely confident. Among passed recordings, higher SQA quality goes with lower uncertainty (partial ρ −0.112 [−0.166, −0.052]).
- **Ground truth agrees.** On the 113 human-Poor recordings the gate let through, murmur entropy is also higher (+0.057 [+0.001, +0.115]), with no clear change in the overconfidence gap.
- **Outcome.** Borderline and high recordings have the same accuracy (0.555 vs 0.563) and the same entropy (0.912 vs 0.911). A weak correlation remains (partial ρ −0.055).
- **Not blind, but different.**
  - Downstream entropy identifies SQA-rejected recordings at AUROC 0.60 (murmur) and 0.57 (outcome).
  - The rejected recordings overlap the most uncertain ones only 1.18× and 1.13× more than chance.
  - At patient level, the two gates' deferrals overlap no more than chance.
- **Gate on vs off (patient level):**
  - The gate leaves 82.6 % of labelled murmur patients with a usable recording.
  - For murmur it improves calibration (ECE 0.065 → 0.040), F1 (+0.019), accuracy (+0.012) and specificity (+0.016).
  - The calibration gain is also present on the same patients, so it comes from dropping poor recordings within each patient.
  - Compared with random rejection of the same share, it keeps murmur sensitivity +0.048 higher.
  - For outcome, specificity rises by +0.047.

## Findings

1. **For murmur, the model's confidence reflects the quality risk left after the SQA gate, and honestly.** Borderline and human-Poor recordings that pass get higher uncertainty, without false confidence.
2. **For outcome, quality does not change accuracy among accepted recordings,** so there is nothing for uncertainty to reflect.
3. **SQA and uncertainty capture largely different information.** The SQA gate adds something the confidence gate does not have.
4. **Keep both gates.** The SQA gate improves murmur calibration on its own. RQ6 combines the two gates at a fixed workload.

For the full tables with confidence intervals, the secondary measures, the gate-on/off comparisons and the notes on interpretation, see the [report](MySteth2_RQ1_SQA_Uncertainty.pdf).
