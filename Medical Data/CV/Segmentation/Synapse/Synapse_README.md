# Synapse Abdomen Dataset Preprocessing

This repository preprocesses the Synapse Abdomen dataset in a way that is compatible with segmentation pipelines such as TransUNet. The preprocessing is implemented directly in the notebook and follows the same general strategy as the dataset preparation used in the TransUNet project: [TransUNet](https://github.com/Beckschen/TransUNet/tree/main/datasets)

## Overview

The raw dataset is loaded from the Abdomen folder, where each patient case contains image volumes and label volumes stored as NIfTI files. The notebook loops through all available cases, performs intensity clipping and normalization, and then saves the results in two separate output formats:

- Training data: 2D slices saved as `.npz` files
- Test data: 3D volumes saved as `.npy.h5` files

The output folder is created as:

```text
Synapse Abdomen Preprocessed/
├── Train/
│   ├── case0001_slice001.npz
│   ├── case0001_slice002.npz
│   └── ...
├── Test/
│   ├── case0001.npy.h5
│   ├── case0002.npy.h5
│   └── ...
├── Synapse Lists/
│   ├── train.txt
│   └── test_vol.txt
└── ...
```

---

## Input Data Structure

The source data comes from a folder organized like this:

```text
Abdomen/
└── RawData/
    └── Training/
        ├── img/
        │   ├── img0001/
        │   │   ├── image_0001.nii.gz
        │   │   ├── image_0002.nii.gz
        │   │   └── ...
        │   └── ...
        ├── label/
        │   ├── label0001/
        │   │   ├── label_0001.nii.gz
        │   │   ├── label_0002.nii.gz
        │   │   └── ...
        │   └── ...
```

For each case:

- `img_path_folder` contains the 3D MRI image slices for one patient
- `label_path_folder` contains the matching segmentation label slices for the same patient
- The code expects the number of image files and label files to match exactly

The notebook asserts this with:

```python
assert len(img_list) == len(label_list)
```

This ensures each image slice has a corresponding label slice.

---

## Case Selection

The notebook processes a predefined list of case IDs:

```python
all_lst = [
    "0031", "0007", "0009", "0005", "0026", "0039", "0024", "0034",
    "0033", "0030", "0023", "0040", "0010", "0021", "0006", "0027",
    "0028", "0037", "0008", "0022", "0038", "0036", "0032", "0002",
    "0029", "0003", "0001", "0004", "0025", "0035"
]
```

The test set is explicitly defined as:

```python
test = ["0008", "0022", "0038", "0036", "0032", "0002", "0029", "0003", "0001", "0004", "0025", "0035"]
```

Any case in `test` is treated as a full-volume test case. All other selected cases are split into 2D training slices.

---

## Preprocessing Steps Per Volume

For every image/label pair, the notebook performs the following:

### 1. Load the NIfTI volume

```python
load_img = nib.load(img_path)
load_label = nib.load(label_path)
```

- `load_img.get_fdata()` returns the image volume as a NumPy array
- `load_label.get_fdata()` returns the label volume as a NumPy array

### 2. Clip the intensity range

```python
low, high = -125, 275
vol_img = load_img.get_fdata().astype(np.float32)
vol_label = load_label.get_fdata().astype(np.int16)
vol_img = np.clip(vol_img, low, high)
```

This is important because CT scans often contain extreme outliers. Clipping keeps the range meaningful and stable for model training.

- `vol_img` is clipped to the interval `[-125, 275]`
- `vol_label` is kept as integer labels, not normalized
- `vol_label` keeps the original label values from the dataset, which are semantic segmentation target values

### 3. Verify shape consistency

```python
assert vol_img.shape == vol_label.shape
```

This guarantees the image volume and segmentation label volume have matching spatial dimensions.

### 4. Normalize the image to [0, 1]

```python
vol_img = (vol_img - low) / (high - low)
```

This converts the CT intensity range to a normalized scale in the range `[0, 1]`.

At this point:

- `vol_img` is a float array in `[0, 1]`
- `vol_label` remains integer-labeled segmentation map

---

## Output Data Format

The dataset is split into two output categories depending on whether the case is in the training or test set.

### Training Output: `.npz` slices

Training data are converted from 3D volumes into 2D slices.

```python
def save_slices_npz(vol_img, vol_label, case_id, train_dir, train_list):
    H, W, D = vol_img.shape

    for sl in range(D):
        img2d = vol_img[:, :, sl]
        lab2d = vol_label[:, :, sl]

        slice_name = f"case{case_id}_slice{(sl+1):03d}"
        fname = os.path.join(train_dir, slice_name)

        np.savez(fname,
                image = img2d.astype(np.float32),
                label = lab2d.astype(np.uint8))

        train_list.append(slice_name)
```

This means each training sample is saved as a single `.npz` file containing exactly two arrays:

- `image`: 2D grayscale CT slice
- `label`: 2D segmentation mask

#### Training sample shape

For a normal training slice:

- `image.shape == (512, 512)`
- `label.shape == (512, 512)`

#### Training array dtypes

- `image.dtype == np.float32`
- `label.dtype == np.uint8`

#### What each key stores

- `image`: a 2D NumPy array of shape `(H, W)` with normalized CT intensity values in `[0, 1]`
- `label`: a 2D NumPy array of shape `(H, W)` with integer segmentation labels for the organs

#### Example loading for training data

```python
import numpy as np

path = "Synapse Abdomen Preprocessed/Train/case0001_slice001.npz"
data = np.load(path)

img = data["image"]
label = data["label"]

print(type(img), img.dtype, img.shape)
print(type(label), label.dtype, label.shape)
```

Typical output:

```python
<class 'numpy.ndarray'> float32 (512, 512)
<class 'numpy.ndarray'> uint8 (512, 512)
```

This is the standard training sample format used by segmentation models. Each `.npz` file is a dictionary-like NumPy archive, where the keys are `image` and `label`.

---

### Test Output: `.npy.h5` volumes

Test cases are kept as full 3D volumes instead of slicing them into 2D samples.

```python
def save_case_h5(vol_img, vol_label, case_id, test_dir):
    fname = os.path.join(test_dir, f"case{case_id}.npy.h5")

    with h5py.File(fname, "w") as f:
        f.create_dataset("image", data=vol_img.astype(np.float32))
        f.create_dataset("label", data=vol_label.astype(np.uint8))
```

This saves a single HDF5 file per test volume containing:

- `image`: a 3D CT volume
- `label`: a 3D segmentation volume

#### Test sample shape

For a test case:

- `image.shape == (H, W, Slices)`
- `label.shape == (H, W, Slices)`

In this project, the common size is:

```text
(H, W, Slices) = (512, 512, num_slices)
```

#### Test array dtypes

- `image.dtype == np.float32`
- `label.dtype == np.uint8`

#### What each dataset stores

- `image`: 3D normalized CT volume, shape `(H, W, D)`, dtype `float32`
- `label`: 3D segmentation mask, shape `(H, W, D)`, dtype `uint8`

#### Example loading for test data

```python
import h5py

path = "Synapse Abdomen Preprocessed/Test/case0001.npy.h5"
with h5py.File(path, "r") as f:
    img = f["image"][:]
    label = f["label"][:]

print(type(img), img.dtype, img.shape)
print(type(label), label.dtype, label.shape)
```

Typical output:

```python
<class 'numpy.ndarray'> float32 (512, 512, 128)
<class 'numpy.ndarray'> uint8 (512, 512, 128)
```

The exact third dimension depends on how many slices the case contains.

---

## List Files

The notebook writes text files listing the processed case names:

```python
list_dir = os.path.join(dst, "Synapse Lists")
os.makedirs(list_dir, exist_ok=True)

with open(os.path.join(list_dir, "train.txt"), "w") as f:
    for name in train_list:
        f.write(name + "\n")

with open(os.path.join(list_dir, "test_vol.txt"), "w") as f:
    for name in test_list:
        f.write(name + "\n")
```

The generated list files contain:

- `train.txt`: one training sample identifier per line, such as `case0001_slice001`
- `test_vol.txt`: one test volume identifier per line, such as `case0001`

These files are useful when you want to build a DataLoader or manage dataset splits in a training script.

---

## Summary of Data Types

### Training items

- File extension: `.npz`
- Type: NumPy archive dictionary-like object
- Keys: `image`, `label`
- `image`: `np.ndarray`, shape `(512, 512)`, dtype `float32`
- `label`: `np.ndarray`, shape `(512, 512)`, dtype `uint8`

### Test items

- File extension: `.npy.h5`
- Type: HDF5 file
- Datasets: `image`, `label`
- `image`: `np.ndarray`, shape `(512, 512, num_slices)`, dtype `float32`
- `label`: `np.ndarray`, shape `(512, 512, num_slices)`, dtype `uint8`

---

## Label Information

The dataset retains the original 13 labels from the Abdomen segmentation dataset, and the notebook does not automatically remap them. Therefore, the user must remap labels according to the target segmentation problem.

The recommended mapping from the project is:

```python
label_mapping = {
    0: 0,
    8: 1,   # Aorta
    4: 2,   # Gallbladder
    3: 3,   # Left Kidney
    2: 4,   # Right Kidney
    6: 5,   # Liver
    11: 6,  # Pancreas
    1: 7,   # Spleen
    7: 8,   # Stomach
}
```

This mapping is commonly used for abdominal organ segmentation tasks, but it is not enforced automatically in the preprocessing code. If your training task requires different classes, you can define a different mapping after loading the saved labels.

Example remapping:

```python
import numpy as np

label = data["label"]
remapped = np.zeros_like(label, dtype=np.uint8)

for old_label, new_label in label_mapping.items():
    remapped[label == old_label] = new_label
```

---

## Practical Usage

### Load a training example

```python
import numpy as np

sample = np.load("Synapse Abdomen Preprocessed/Train/case0001_slice001.npz")
image = sample["image"]
label = sample["label"]

print(image.min(), image.max())
print(np.unique(label))
```

This tells you:

- the image intensities are normalized to `[0, 1]`
- the label values contain the raw class IDs, which you may need to remap

### Load a test volume

```python
import h5py

with h5py.File("Synapse Abdomen Preprocessed/Test/case0001.npy.h5", "r") as f:
    image = f["image"][:]
    label = f["label"][:]

print(image.shape, image.dtype)
print(label.shape, label.dtype)
```

This gives you the full 3D case for inference or evaluation.

---

## Final Notes

This preprocessing pipeline intentionally separates the data into:

- 2D slices for training, because many segmentation models train on slices
- 3D volumes for testing, because full volumes are often used during validation or inference

The key point to remember is:

- `image` always stores the normalized CT intensity data
- `label` always stores the segmentation mask
- training examples are saved as `.npz` archives
- test examples are saved as HDF5 files
- both use `float32` for images and `uint8` for labels

This is the core format you should use whenever reading the dataset back into a training or evaluation script.
