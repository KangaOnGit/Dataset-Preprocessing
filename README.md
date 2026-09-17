# Dataset Preprocessing

A repository for preprocessing and organizing datasets used in biomedical and computer vision experiments. The project currently focuses on medical image segmentation datasets, with each dataset prepared into a clean, model-ready format for training, validation, and inference.

## Repository Purpose

This repository stores dataset processing workflows and preprocessed outputs for segmentation tasks. The goal is to take raw public datasets, standardize their structure, and save the results in formats that are easy to load and use in machine learning pipelines.

The project is designed for reproducibility and ease of reuse. Once processed, the data are organized into clear training/testing directories with consistent file naming, data types, and label conventions.

---

## Included Datasets

### 1. Synapse Abdomen

The Synapse Abdomen dataset is processed for abdominal organ segmentation.

Relevant files:

- [medical/cv/segmentation/Synapse/Synapse_README.md](medical/cv/segmentation/Synapse/Synapse_README.md)
- [medical/cv/segmentation/Synapse/Preprocess.ipynb](medical/cv/segmentation/Synapse/Preprocess.ipynb)

Key preprocessing characteristics:

- Training samples are sliced into 2D images of shape `(512, 512)`
- Test samples remain 3D volumes of shape `(512, 512, num_slices)`
- Input intensities are clipped and normalized to `[0, 1]`
- Training outputs are saved as `.npz` files with keys `image` and `label`
- Test outputs are saved as HDF5 files with datasets `image` and `label`
- Original dataset labels are retained and must be remapped for model training

### 2. MoNuSeg

The MoNuSeg dataset is processed for nuclei instance segmentation.

Relevant files:

- [medical/cv/segmentation/MoNuSeg/MoNuSeg_README.md](medical/cv/segmentation/MoNuSeg/MoNuSeg_README.md)
- [medical/cv/segmentation/MoNuSeg/Preprocess.ipynb](medical/cv/segmentation/MoNuSeg/Preprocess.ipynb)

Key preprocessing characteristics:

- Microscopy images are resized to `(1000, 1000)`
- XML polygon annotations are converted into instance masks
- Output images are saved as PNG files
- Output labels are saved as `.npy` arrays
- Masks are instance segmentation maps, where `0` is background and nonzero values represent different nuclei

---

## Repository Structure

```text
Data-Preprocess/
├── LICENSE
├── README.md
├── medical/
│   ├── cv/
│   │   ├── object_detection/
│   │   │   └── (reserved for object detection datasets)
│   │   └── segmentation/
│   │       ├── MoNuSeg/
│   │       │   ├── Preprocess.ipynb
│   │       │   ├── MoNuSeg_README.md
│   │       │   └── monuseg_converted/
│   │       └── Synapse/
│   │           ├── Preprocess.ipynb
│   │           ├── Synapse_README.md
│   │           └── Synapse Abdomen Preprocessed/
│   └── nlp/
│       └── (reserved for NLP datasets and preprocessing pipelines)
```

---

## Typical Dataset Usage

Each dataset is prepared so that it can be used directly in segmentation experiments.

### General pattern

- Images are represented as NumPy arrays or image files
- Labels are represented as mask arrays or dictionary entries
- Training splits are saved separately from testing splits
- File naming is consistent to help match image and label pairs

### Example usage

```python
import numpy as np

# Load Synapse training sample
sample = np.load(
    "medical/cv/segmentation/Synapse/Synapse Abdomen Preprocessed/Train/case0001_slice001.npz"
)
image = sample["image"]
label = sample["label"]

print(image.shape, image.dtype)
print(label.shape, label.dtype)
```

```python
from PIL import Image
import numpy as np

# Load MoNuSeg image and mask
image = np.array(
    Image.open("medical/cv/segmentation/MoNuSeg/monuseg_converted/training/images/image_001.png")
)
mask = np.load(
    "medical/cv/segmentation/MoNuSeg/monuseg_converted/training/labels/image_001.npy"
)

print(image.shape, image.dtype)
print(mask.shape, mask.dtype)
print(np.unique(mask))
```

---

## Notes on Data Handling

- Some datasets retain original source labels and require custom remapping before training.
- Image intensities are sometimes normalized into `[0, 1]` for model stability.
- Segmentation masks may be either semantic masks or instance masks depending on the dataset.
- Always inspect the dataset-specific README before training to confirm label semantics and output structure.

---

## Intended Use

This repository is intended for:

- dataset preparation and preprocessing
- reproducible segmentation experiments
- storing model-ready data in a consistent structure
- documenting label mappings, shapes, and output formats for future use

It is especially useful when returning to older experiments and needing to know exactly how the data were generated and what each file contains.

---

## License

This repository is provided for educational and research use. Please respect the licensing and citation terms of each original dataset source.

---

## Future Scope

This repository can expand to include additional datasets beyond medical segmentation, such as:

- other anatomical segmentation datasets
- cell microscopy datasets
- natural image segmentation datasets
- NLP or text-based dataset preprocessing projects

The structure is designed to scale as more preprocessing workflows are added.
