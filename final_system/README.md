# final_system

This folder holds the outputs of the finished MySteth 2.0 system:

- **its decisions for every patient**, on CirCor (RQ6) and on ZCHSound (RQ7);
- **the settings it runs with**: temperatures and decision thresholds.

The trained networks themselves are in Release [models-v1](https://github.com/Tusharr-Bansal/MySeth-2.0/releases/tag/models-v1) (`mysteth2_models.zip`). The CSVs here are copies of files in RQ4's, RQ6's and RQ7's outputs, gathered in one place.

```
final_system/
├── patient_level/
│   ├── rq6_patient_decisions.csv   ← CirCor, 942 patients (from RQ6)
│   └── rq7_children.csv            ← ZCHSound, 1,259 children (from RQ7)
└── model_config/
    ├── rq4_temperatures.csv        ← temperatures of the system model, per fold and seed
    ├── rq4_thresholds.csv          ← decision thresholds per fold, for every UQ method
    ├── rq6_temperatures.csv        ← temperatures of the system model and the two ablation models
    └── rq7_thresholds.csv          ← one threshold per task, for new data
```

## The final system in one paragraph

A recording first goes through the **SQA gate** (`p_good` ≥ 0.55; the threshold is saved inside the SQA model). A patient with no passing recording is asked to **re-record**. For everyone else, the **system model** (the shared Domain-SSL encoder with murmur and outcome heads, 3 folds × seeds 42, 123 and 7) scores every 5 s window. The 3 seeds' probabilities are temperature-scaled and averaged (`ens_temp`), windows are averaged per recording, and recordings are combined per patient (murmur: the highest, outcome: the mean). Each task's decision is probability ≥ threshold, and its **Chow risk** is p / t below the threshold and (1 − p) / (1 − t) above it. The **confidence gate** turns each task's risk into a percentile among the patients who reach it, takes the higher of the two, and sends the top **20 %** to a **clinician**. Everyone else gets an automated decision for both tasks, explained with Grad-CAM.

On CirCor this gives: **22.1 %** re-record, **15.6 %** clinician, **62.3 %** automated (942 patients).

## patient_level/

### `rq6_patient_decisions.csv`: CirCor

One row per **configuration × task × patient**: 9 configurations × 2 tasks × 942 patients = 16,956 rows. All probabilities are out of fold: each patient is scored by the networks of the fold in which it is a test patient (see [`shared/`](../shared/)).

| Column | Meaning |
|---|---|
| `row` | The configuration (see the next table). Filter on `row == "full system"` for the final system. |
| `task` | `murmur` or `outcome`. |
| `patient_id`, `group`, `fold` | CirCor patient, repeat-visit group and test fold (as in `shared/rq2_folds.csv`). |
| `murmur`, `outcome` | CirCor labels. |
| `y` | The label for this task as a number: murmur Present = 1, Absent = 0, Unknown = −1; outcome Abnormal = 1, Normal = 0. |
| `p` | Patient probability for this task. Empty for patients with no recording that passed SQA, in configurations with the SQA gate. |
| `t` | Decision threshold of the patient's fold. |
| `risk` | Chow risk from `p` and `t`. |
| `yhat` | Decision: 1 if `p` ≥ `t`, else 0. Empty for re-recorded patients. |
| `covered` | `True` if at least one of the patient's recordings passed SQA (always `True` without the SQA gate). |
| `deferred` | `True` if the confidence gate sent the patient to a clinician. The gate is per patient, so this is the same in both task rows. |
| `auto` | `True` if the decision is automated: `covered` and not `deferred`. |

A patient is **re-recorded** if `covered` is `False`, goes to a **clinician** if `deferred` is `True`, and is **automated** if `auto` is `True`.

| `row` | What changes compared with the full system |
|---|---|
| `full system` | Nothing: SQA gate, system model, `ens_temp`, confidence gate (20 %). |
| `− SQA gate` | No SQA gate: every patient reaches the confidence gate, which defers 20 % of them. |
| `− SQA gate, same total workload` | No SQA gate, but the confidence gate defers as many patients as the full system re-records and defers together (37.7 %). |
| `− SSL` | Joint model trained from scratch instead of from the Domain-SSL encoder. |
| `− multi-task` | RQ2's two separate single-task Domain-SSL models instead of the shared one. |
| `− UQ` | One network (seed 42) with no temperature scaling, instead of `ens_temp`. |
| `− confidence gate` | No confidence gate: every patient who passes SQA is decided automatically. |
| `− XAI` | No Grad-CAM. XAI does not change decisions, so this row is the same as the full system. |
| `floor` | The baseline: one network (seed 42), no SQA gate, no confidence gate. |

The 68 patients with an `Unknown` murmur have `y = −1` in the murmur rows. They have decisions, but no murmur answer to check them against.

### `rq7_children.csv`: ZCHSound

One row per child (one recording each): 1,259 rows. Nothing was trained or tuned on ZCHSound; every setting comes from CirCor.

| Column | Meaning |
|---|---|
| `fileName`, `gender`, `age(days)`, `diagnosis` | From ZCHSound's label files, as given (one `gender` entry is `F ` with a trailing space). `diagnosis`: NORMAL, ASD, VSD, PFO or PDA. |
| `subset` | ZCHSound's own split: `clean` (941) or `noisy` (318). |
| `chd` | 1 if `diagnosis` is ASD, VSD, PFO or PDA; 0 if NORMAL. |
| `duration_s` | Recording length in seconds. |
| `p_good`, `sqa_pass` | SQA probability of a good recording, and whether it passes (≥ 0.55). |
| `p_murmur`, `pred_murmur` | Murmur probability and decision (threshold from `rq7_thresholds.csv`). |
| `p_outcome`, `pred_outcome` | Outcome probability and decision. |
| `risk_murmur`, `risk_outcome` | Chow risk per task. |
| `full_covered` | `True` if the recording passed SQA (same as `sqa_pass`). |
| `full_deferred` | `True` if the confidence gate sent the child to a clinician (20 % of the children who passed SQA). |

Probabilities and decisions are filled in for every child, including those who failed SQA. In the full system those children are re-recorded, so use `full_covered` and `full_deferred` to get the system's actual decision: re-record if `full_covered` is `False`, clinician if `full_deferred` is `True`, automated otherwise.

## model_config/

| File | What it holds | Used by the final system |
|---|---|---|
| `rq4_temperatures.csv` | Per task, fold and seed (18 rows): `T_deterministic` and `T_mc`. | `T_deterministic` (the temperature in `ens_temp`). `T_mc` is only for RQ4's MC-dropout methods. |
| `rq6_temperatures.csv` | Per model, task, fold and seed (54 rows), for `joint_domain` (the system model), `joint_scratch` (− SSL) and `separate` (− multi-task). | The `joint_domain` rows, which equal `T_deterministic` above. |
| `rq4_thresholds.csv` | Per task, UQ method and fold (42 rows): `threshold_patient` and `threshold_recording`. | The `ens_temp` rows, `threshold_patient`. These are the `t` values in `rq6_patient_decisions.csv`. |
| `rq7_thresholds.csv` | One threshold per task: murmur 0.407, outcome 0.413. | For new data. Each is the mean of the 3 folds' `ens_temp` patient thresholds. |

Temperatures were fitted on each fold's `cal` recordings; thresholds were chosen on each fold's `val` patients (murmur: weighted accuracy; outcome: the CirCor cost).

### Using the system on new recordings (as in RQ7)

1. Score each recording with the SQA model (`sqa_final_model.joblib`); a child with no recording at `p_good` ≥ 0.55 is re-recorded.
2. Run all 9 networks on 5 s windows (2.5 s hop) of the 4 kHz audio.
3. For each fold, divide each seed's logit (class 1 minus class 0) by its `joint_domain` temperature, take the sigmoid, and average the 3 seeds; average the windows of a recording; then average the 3 folds.
4. Combine recordings per child (murmur: highest, outcome: mean) and compare with `rq7_thresholds.csv`.
5. Compute each task's Chow risk; among the children who passed SQA, rank by the higher of the two risk percentiles and send the top 20 % to a clinician.
