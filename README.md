# Dataset Preprocessing

This repository contains reproducible preprocessing workflows for biomedical datasets used in computer vision and segmentation experiments.

Each dataset is converted into a consistent, model-ready structure and documented separately in its own README. Raw datasets are not stored in this repository.

## Repository Structure

```text
Data-Preprocess/
├── README.md
├── LICENSE
└── medical/
    ├── cv/
    │   ├── segmentation/
    │   │   ├── Synapse/
    │   │   ├── MoNuSeg/
    │   │   ├── GlaS/
    │   │   └── EBHI (Adenocarcinoma)/
    │   └── object_detection/
    └── nlp/
```

## Segmentation Datasets

### Synapse Abdomen

[Synapse dataset README](medical/cv/segmentation/Synapse/Synapse_README.md)

Notebooks:

- `preprocess_1.ipynb` converts NIfTI volumes into normalized 2D training slices and 3D test volumes.
- `preprocess_2.ipynb` splits the training cases by case into approximately 80% training and 20% validation.

Output includes:

- 2D `.npz` training and validation slices
- 3D `.npy.h5` test volumes
- Optional reconstructed validation volumes
- `train.txt`, `val.txt`, `val_vol.txt`, and `test_vol.txt`

### MoNuSeg

[MoNuSeg dataset README](medical/cv/segmentation/MoNuSeg/MoNuSeg_README.md)

Notebooks:

- `preprocess_1.ipynb` converts TIFF images and XML polygon annotations into PNG images and instance masks.
- `preprocess_2.ipynb` provides an optional patient-level 80/20 training/validation split.

The MoNuSeg training set is small. For routine experiments, the recommended approach is to:

- run only `preprocess_1.ipynb`;
- train on the complete official training set; and
- use k-fold cross-validation, or evaluate directly on the official test set.

The fixed validation split can be useful for experiments, but should be used with care because a validation subset may not contain representative tissue coverage.

### GlaS

[GlaS dataset README](medical/cv/segmentation/GlaS/GlaS_README.md)

`preprocess.ipynb` copies the original BMP images and masks into training, validation, Test A, Test B, and combined Test A/B folders. Training images are deterministically split into approximately 80% training and 20% validation, and masks receive a `_mask` suffix.

Output includes:

- `train/images` and `train/masks`
- `val/images` and `val/masks`
- `testA/images` and `testA/masks`
- `testB/images` and `testB/masks`
- `testAB/images` and `testAB/masks`

### EBHI-SEG Adenocarcinoma

[EBHI Adenocarcinoma dataset README](<medical/cv/segmentation/EBHI (Adenocarcinoma)/EBHI_Adenocarcinoma_README.md>)

`preprocess.ipynb` pairs source images with labels, creates reproducible training, validation, and test splits, copies both modalities into each split, and adds `_mask` to mask filenames.

The default split is approximately:

- 72.25% training
- 12.75% validation
- 15% test

## General Output Conventions

The preprocessing workflows generally follow these conventions:

- images and masks are stored in separate directories;
- matching samples use a common filename stem;
- masks may include a configurable suffix such as `_mask`;
- training, validation, and test splits are separated;
- original label values and formats are preserved unless the dataset README states otherwise;
- split metadata or list files are written when required by the downstream training pipeline.

Always inspect the dataset-specific README before training because formats, label semantics, and validation recommendations differ by dataset.

## Output Formats

| Dataset | Images | Masks/labels |
|---|---|---|
| Synapse | 2D `.npz` slices or 3D `.npy.h5` volumes | Included with each `.npz` or `.npy.h5` file |
| MoNuSeg | `.png` images, resized to `1000 x 1000` | `.npy` instance masks |
| GlaS | `.bmp` images | `.bmp` masks with `_mask` suffix |
| EBHI Adenocarcinoma | PNG, JPG, JPEG, or BMP images | Matching masks with `_mask` suffix and preserved extension |

## Validation Guidance

- Split Synapse data by case to avoid leakage between slices belonging to the same case.
- Keep the full MoNuSeg training set unless an experiment specifically requires a fixed holdout; otherwise use k-fold validation.
- Verify class or tissue coverage when creating smaller validation splits.
- Use the official test split for final evaluation unless the dataset documentation defines another protocol.

## Data Handling Notes

- Source datasets may require separate acceptance of their licenses or registration on their providers' websites.
- Clipping, normalization, resizing, and label remapping are dataset-specific and are documented in each README.
- Some datasets are mask-based, while MoNuSeg uses instance segmentation IDs.
- Rerunning a notebook may recreate or overwrite the generated output directory.

## Intended Use

This repository supports:

- reproducible dataset preparation;
- biomedical segmentation experiments;
- consistent image-mask pairing;
- dataset-specific split documentation;
- reuse of preprocessed datasets across training pipelines.

## License

Please follow the license and citation requirements of each original dataset. The license for this repository is available in [LICENSE](LICENSE).
