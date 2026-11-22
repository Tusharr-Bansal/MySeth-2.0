# shared

This folder has one file, `rq2_folds.csv`. It sets how CirCor is split into folds for the whole study. Every model from RQ2 onward was trained, tuned and tested on these folds, so a patient is in the same test fold in RQ2, RQ3, RQ4, RQ5, RQ6 and RQ7. RQ1 also uses these folds, through RQ4's saved scores. The SQA model in `00_SQA` uses its own cross-validation split instead.

The file was created by [RQ2 part a](../RQ2/code/rq2-part-1-again.ipynb), section 1. RQ2 part b and RQ3 load it and save an identical copy in their outputs, and RQ4–RQ7 read it from RQ3's output. All three copies are the same file.

## What is in it

One row per CirCor recording: **3,163 recordings from 942 patients**.

| Column | Meaning |
|---|---|
| `wid` | Recording ID: `<patient>_<site>`, e.g. `13918_AV`. When a patient has more than one recording at a site, a number is added (`49952_MV_1`). It matches the `.wav` file name in CirCor. |
| `patient_id` | CirCor patient ID. |
| `group` | Patient group. Children who came back for a second visit appear in CirCor under two patient IDs, linked by `Additional ID`. Those IDs are put in one group so that both visits always land in the same fold. The group ID is one of the linked patient IDs. 942 patients make 872 groups (70 groups hold two patient IDs). |
| `location` | Auscultation site: `AV`, `MV`, `PV`, `TV`, or `Phc` (4 recordings). |
| `murmur` | Murmur label from CirCor: `Present`, `Absent` or `Unknown`. |
| `outcome` | Outcome label from CirCor: `Normal` or `Abnormal`. |
| `y_murmur` | Murmur as a number: Present = 1, Absent = 0, Unknown = −1. |
| `y_outcome` | Outcome as a number: Abnormal = 1, Normal = 0. |
| `fold` | The fold (0, 1 or 2) in which this recording is a **test** recording. |
| `role_f0`, `role_f1`, `role_f2` | What the recording is used for when that fold is the test fold: `train`, `val`, `cal` or `test`. |

Labels are per patient, so every recording of a patient carries the same `murmur` and `outcome`.

## How the folds were made

- **Folds:** 3 folds, split by `group`, and balanced on murmur × outcome (`StratifiedGroupKFold`, seed 42).
- **Roles inside each fold:** the fold's own recordings are `test`. The groups in the other two folds are split, balanced on murmur × outcome, into `train` (75%), `val` (10%) and `cal` (15%).
- **No leakage:** a group has exactly one fold and exactly one role in each fold, so no child, and no second visit of a child, is ever on both sides of a split.

| Fold | Test recordings | Test patients |
|---|---|---|
| 0 | 1,065 | 315 |
| 1 | 1,061 | 313 |
| 2 | 1,037 | 314 |

When fold *f* is tested, the other two folds give roughly 472 `train`, 62 `val` and 94 `cal` patients.

## What each role is used for

| Role | Used for |
|---|---|
| `train` | Fitting the network. |
| `val` | Early stopping, and choosing each fold's decision thresholds (murmur: weighted accuracy; outcome: the CirCor cost). |
| `cal` | Fitting the temperature for temperature scaling (`ens_temp`). |
| `test` | Out-of-fold results: each recording is scored by the models of the fold in which it is `test`, which never saw it during training, early stopping, thresholding or calibration. |

The SSL pretraining in RQ2 part a is the one exception to these roles: for fold *f* it uses every recording outside fold *f* (`train`, `val` and `cal`), with no labels. Test recordings are never seen.

## Unknown murmurs

68 patients (156 recordings) have murmur `Unknown`: clinicians could not call it. They stay in the folds. During training they count only toward the outcome loss, and they are left out of murmur accuracy metrics because they have no murmur answer to check against.

## Reading it

```python
import pandas as pd
folds = pd.read_csv("shared/rq2_folds.csv")

test = folds[folds["role_f0"] == "test"]        # same rows as folds[folds["fold"] == 0]
train = folds[folds["role_f0"] == "train"]
```
