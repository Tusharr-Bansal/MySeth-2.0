## 1. Datasets (not in this repo)
- **CirCor DigiScope Phonocardiogram Dataset** v1.0.3 (audio, patient metadata, segmentation .tsv), ODC-By 1.0
  - Original: https://physionet.org/content/circor-heart-sound/1.0.3/
  - Kaggle copy used: https://www.kaggle.com/datasets/bjoernjostein/the-circor-digiscope-phonocardiogram-dataset-v2
- **ZCHSound** (Jia et al., IEEE TBME 2024)
  - Original: http://zchsound.ncrcch.org.cn/
  - Kaggle upload used: https://www.kaggle.com/datasets/tusharrbb/zchsounds
- **PhysioNet/CinC Challenge 2016**, training set (RQ2 part a pretraining)
  - Original: https://physionet.org/content/challenge-2016/1.0.0/
  - Kaggle copy used: https://www.kaggle.com/datasets/bjoernjostein/physionet-challenge-2016
- **PASCAL Classifying Heart Sounds Challenge** (RQ2 part a pretraining)
  - Kaggle copy used: https://www.kaggle.com/datasets/kinguistics/heartbeat-sounds

## 2. Pretrained models downloaded at run time (SQA)
- PANNs CNN14 (Cnn14_mAP=0.431): https://zenodo.org/records/3987831
- AST: https://huggingface.co/MIT/ast-finetuned-audioset-10-10-0.4593
- CLAP: https://huggingface.co/laion/clap-htsat-unfused
- HeAR: https://huggingface.co/google/hear (requires accepting Google's terms on Hugging Face)

## 3. Kaggle inputs not copied
- SQA embedding cache: https://www.kaggle.com/datasets/tusharrbb/embeddings-sqa
- First RQ2 run (its Scratch arm is reused by RQ2 part b): https://www.kaggle.com/code/tusharrbb/rq2-redone/output

## 4. Outputs kept on Kaggle (Output tab of each notebook; per-file list in LARGE_FILES.md)
- SQA: https://www.kaggle.com/code/tusharrbb/sqa-model-redone/output
- RQ1: https://www.kaggle.com/code/tusharrbb/rq1-version-3/output
- RQ2 part a (6 BYOL-A encoders): https://www.kaggle.com/code/tusharrbb/rq2-part-1-again/output
- RQ2 part b (models, units, frozen embeddings): https://www.kaggle.com/code/tusharrbb/rq2-part-2-again/output
- RQ3 (all four models' weights, units): https://www.kaggle.com/code/tusharrbb/rq3-redone/output
- RQ4 (MC-Dropout passes): https://www.kaggle.com/code/tusharrbb/rq4-redone/output
- RQ5 (per-window explanation maps): https://www.kaggle.com/code/tusharrbb/rq5-version-3/output
- RQ6: https://www.kaggle.com/code/tusharrbb/rq6-redone/output
- RQ7: https://www.kaggle.com/code/tusharrbb/rq7-external/output

The system model (9 networks) and the final SQA model are in this repo's Release `models-v1`.
