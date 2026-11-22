# large_files

Some files are too large for GitHub or are public datasets, so they are not in this repo. This folder lists them, one file per folder of the repo, with links to where they are kept. Most stay on Kaggle in the output of the notebook that made them.

## The files

| File | Notebook(s) on Kaggle | Large outputs kept on Kaggle |
|---|---|---|
| [`00_SQA_Large_Files.md`](00_SQA_Large_Files.md) | `sqa-model-redone` | none |
| [`RQ1_Large_Files.md`](RQ1_Large_Files.md) | `rq1-version-3` | none |
| [`RQ2_Large_Files.md`](RQ2_Large_Files.md) | `rq2-part-1-again`, `rq2-part-2-again` | 192 files, ≈ 2.9 GB: the 6 SSL encoders (part a); the networks and per-unit outputs of all five encoder arms, and the frozen embeddings of the two SSL encoders (part b) |
| [`RQ3_Large_Files.md`](RQ3_Large_Files.md) | `rq3-redone` | 36 files, ≈ 0.8 GB: the 18 joint networks (Domain-SSL and Scratch) and their per-unit outputs |
| [`RQ4_Large_Files.md`](RQ4_Large_Files.md) | `rq4-redone` | 6 files, ≈ 23 MB: MC-dropout passes for seeds 123 and 7 (seed 42's are in RQ3's units) |
| [`RQ5_Large_Files.md`](RQ5_Large_Files.md) | `rq5-version-3` | 9 files, ≈ 0.2 GB: per-window Grad-CAM, gradient × input and phase-SHAP for every unit |
| [`RQ6_Large_Files.md`](RQ6_Large_Files.md) | `rq6-redone` | none |
| [`RQ7_Large_Files.md`](RQ7_Large_Files.md) | `rq7-external` | 1,264 files, ≈ 0.5 GB: a copy of the ZCHSound audio (1,259 `.wav` files) and the 5 SQA-encoder embeddings of ZCHSound |

"None" means every output of that notebook is in this repo, in the folder's `outputs/`.

A **unit** is one network: one fold × one seed (3 folds × seeds 42, 123 and 7).

## What each file contains

Each file has one section per notebook, with:

1. **Kaggle:** the link to the notebook (code, logs and outputs as run).
2. **Inputs used:** every input attached to the notebook, read from its Kaggle settings:
   - public datasets, with the original source next to the Kaggle copy (CirCor and PhysioNet 2016 on PhysioNet, ZCHSound on its own site);
   - Kaggle datasets made for this project (`tusharrbb/...`);
   - outputs of earlier notebooks;
   - pretrained models downloaded while the notebook runs (PANNs, AST, CLAP, HeAR).
3. **Outputs kept on Kaggle:** the link to the notebook's Output tab, and a table of every output file that is not in this repo, with its size in MB.

## Before downloading from here

- **The final models are already on GitHub.** The 9 system-model networks (`joint_domain_f{0,1,2}_s{42,123,7}.pt`, listed in `RQ3_Large_Files.md`) and the SQA model are in Release [models-v1](https://github.com/Tusharr-Bansal/MySeth-2.0/releases/tag/models-v1) (`mysteth2_models.zip`). You only need Kaggle for the comparison arms (RQ2, RQ3 Scratch) and the per-unit files.
- **ZCHSound audio:** use the dataset link in `RQ7_Large_Files.md` rather than the copy in RQ7's output.
- **RQ2 part b** also reads the output of an earlier run, `rq2-redone`, to reuse its Scratch arm. That notebook is not in this repo; its link is in `RQ2_Large_Files.md`.
- **HeAR** needs Google's terms to be accepted on Hugging Face before it downloads.
- Kaggle links open only for notebooks and datasets that are public on Kaggle.
