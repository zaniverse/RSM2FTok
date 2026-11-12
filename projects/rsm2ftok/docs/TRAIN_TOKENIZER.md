# Tokenizer Training and Reconstruction Evaluation

Tokenizer training uses the Detectron2/Mask2Former implementation in the repository root.

## Dataset format

The remote-sensing reconstruction datasets use this layout:

```text
/home/disk2/zan/datasets_seg/<DATASET>/<dataset_name>/
├── train/
│   ├── image/
│   ├── mask/
│   └── mask255/
└── test/
    ├── image/
    ├── mask/
    └── mask255/
```

For every image, `mask255/` contains one binary mask per semantic class. The mask is a PNG with background `0` and foreground `255`; the class id is encoded by the filename suffix:

```text
image/bsb_test_1.png
mask/bsb_test_1.png
mask255/bsb_test_1_c1.png
mask255/bsb_test_1_c2.png
```

The `cN` suffix is the class id (`c1` means class 1). The original semantic mask is converted into these per-class `0/255` masks before training. The `mask/` directory is retained as the source semantic annotation, while the reconstruction loader reads `image/` and `mask255/`.

The current registration file is [register_rs_ch4_recon.py](../../../mask2former/data/datasets/register_rs_ch4_recon.py). Its dataset root is `/home/disk2/zan/datasets_seg`; update `_ROOT` and the dataset-specific paths there when using another layout.

Registered dataset names are generated as `<dataset>_recon_train` and `<dataset>_recon_test` for:

```text
oem, chn6cug, dgroad, ottawa, lrsny, whumix, dglc,
landcoverai, postdam, dlrsd, flair, loveda, earthvl, bsb, isaid
```

The config selects the train/test datasets in `DATASETS.TRAIN` and `DATASETS.TEST`.

## Mapper

The config value:

```yaml
INPUT:
  DATASET_MAPPER_NAME: "mask_former_rs15_recon_ch4"
```

is handled in `train_net_hint.py`. It selects `SemanticB4DatasetMapper` from [mask_former_semantic_b4_dataset_mapper.py](../../../mask2former/data/dataset_mappers/mask_former_semantic_b4_dataset_mapper.py). There is no mapper class named `mask_former_rs15_recon_ch4`; it is the configuration name for this mapper branch.

For each sample, the mapper loads the RGB image, loads `mask255/<stem>_cN.png`, converts `0/255` to a binary mask, applies augmentations, and appends the mask as the fourth hint channel.

## Train

```bash
python train_net_hint.py \
  --config-file projects/rsm2ftok/configs/rsseg/m2ftok_swinl_ch3_hintattn_256x2_640k.yaml \
  --num-gpus 2 \
  --resume \
  OUTPUT_DIR test_output/tokenizer_swinl_640k_work_dir
```

The tokenizer uses the Swin-L initialization checkpoint:

```text
https://github.com/SwinTransformer/storage/releases/download/v1.0.0/swin_large_patch4_window12_384_22k.pth
```

Download the `.pth` file from the link above and save it as `pretrained_weights/swin_large_patch4_window12_384_22k.pth`. The default `MODEL.WEIGHTS` value already points to this path. The download directory contains [pretrained_weights/README.txt](../../../pretrained_weights/README.txt). Set the dataset paths and selected datasets before launching.

## Reconstruction evaluation

Download the released tokenizer checkpoint into the repository:

```bash
bash tools/download_tokenizer_weights.sh
```

The checkpoint is downloaded from [koreyoshieo/RSM2FTok](https://huggingface.co/koreyoshieo/RSM2FTok) to:

```text
pretrained_weights/tokenizer/rsm2ftok_256x2_640k.pth
```

Run reconstruction evaluation with:

```bash
bash tools/eval_tokenizer.sh
```

To select GPUs and an output directory:

```bash
NUM_GPUS=2 OUTPUT_DIR=test_output/rsm2ftok_256x2_640k_eval bash tools/eval_tokenizer.sh
```

The evaluator is `ReconEvaluator` in [mask_recon_evaluator.py](../../../mask2former/evaluation/mask_recon_evaluator.py). It reports `Recon_IoU`, `Recon_Acc`, and `Recon_BoundaryIoU`; `Recon_Acc` is foreground recall (`intersection / ground-truth foreground area`).
