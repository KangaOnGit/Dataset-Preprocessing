# EBHI Adenocarcinoma Dataset Preprocessing

This preprocessing pipeline converts the EBHI-SEG adenocarcinoma subset into a clean, model-ready structure. The notebook:

1. Locates the source image and label directories
2. Filters supported image formats from the source directory
3. Splits the images into training, validation, and test subsets
4. Copies each image and its matching mask into the appropriate split
5. Adds a `_mask` suffix to mask filenames

The default output is organized as:

```text
processed_ebhi/
├── train/
│   ├── images/
│   └── masks/
├── val/
│   ├── images/
│   └── masks/
└── test/
    ├── images/
    └── masks/
```

---

## Dataset Source

The notebook is configured for the Kaggle dataset:

```python
BASE_SRC = "/kaggle/input/datasets/alibabaei78/ebhi-seg/EBHI-SEG/Adenocarcinoma"
SRC_IMG_DIR = os.path.join(BASE_SRC, "image")
SRC_MASK_DIR = os.path.join(BASE_SRC, "label")
DEST_DIR = "processed_ebhi"
```

Update `BASE_SRC` when running the notebook outside Kaggle. The expected source structure is:

```text
Adenocarcinoma/
├── image/
│   ├── sample_001.png
│   ├── sample_002.png
│   └── ...
└── label/
    ├── sample_001.png
    ├── sample_002.png
    └── ...
```

The image and mask filenames in the two source directories are expected to match exactly.

---

## Dataset Split

The notebook creates a deterministic three-way split:

```python
TEST_SIZE = 0.15
VAL_SIZE = 0.15
SEED = 42
```

The split is performed in two stages:

1. 15% of all images is assigned to the test set.
2. 15% of the remaining dataset is assigned to the validation set.

```python
train_val_files, test_files = train_test_split(
    all_files,
    test_size=TEST_SIZE,
    random_state=SEED
)

relative_val_size = VAL_SIZE / (1 - TEST_SIZE)
train_files, val_files = train_test_split(
    train_val_files,
    test_size=relative_val_size,
    random_state=SEED
)
```

Because `train_test_split` uses one common seed for both stages, the resulting split is reproducible when the source file ordering and dataset contents remain unchanged.

For a dataset containing `N` images:

| Split | Approximate proportion |
|---|---:|
| Training | 72.25% |
| Validation | 12.75% |
| Test | 15.00% |

The final counts depend on `train_test_split` rounding for the available number of images.

---

## Supported Input Formats

The notebook scans the source image directory and retains only common raster formats:

```python
all_files = [f for f in all_files if f.lower().endswith(
    ('.png', '.jpg', '.jpeg', '.bmp')
)]
```

Supported extensions are:

- `.png`
- `.jpg`
- `.jpeg`
- `.bmp`

Mask files must use the same filename and extension as their matching images.

---

## Image and Mask Matching

For each assigned image, the notebook constructs the mask path by keeping the same filename:

```python
src_img_path = os.path.join(SRC_IMG_DIR, filename)
src_mask_path = os.path.join(SRC_MASK_DIR, filename)
```

If a mask is missing, that sample is skipped with a warning.

For example:

```text
image/sample_001.png
label/sample_001.png
```

The mask is copied to the split while its stem receives the configured suffix:

```python
NEW_MASK_SUFFIX = "_mask"
```

This produces:

```text
train/images/sample_001.png
train/masks/sample_001_mask.png
```

The same behavior supports `.jpg`, `.jpeg`, and `.bmp` files because the original extension is preserved.

---

## Preprocessing Steps

### 1. Validate the source directories

The notebook stops immediately if either required directory is missing:

```python
if not os.path.exists(SRC_IMG_DIR):
    raise FileNotFoundError(...)

if not os.path.exists(SRC_MASK_DIR):
    raise FileNotFoundError(...)
```

### 2. Recreate the output directory

If `processed_ebhi` already exists, the notebook removes it before processing:

```python
if os.path.exists(DEST_DIR):
    shutil.rmtree(DEST_DIR)
```

This ensures that stale files from a previous run do not remain in the output.

### 3. Create split directories

The notebook creates separate `images` and `masks` directories for:

- `train`
- `val`
- `test`

### 4. Create the split

Supported source images are sorted before splitting. The notebook uses scikit-learn for reproducible train/validation/test partitioning.

### 5. Copy paired samples

For each split, the notebook uses `shutil.copy2()` to copy the image and mask. The image content is unchanged, and the mask content and extension are also unchanged.

The notebook does not:

- resize images
- normalize pixel values
- convert color spaces
- modify mask values
- change file formats

Only directory placement and mask filenames are changed.

---

## Output Format

Images retain their original filenames and formats.

Masks retain their original formats and receive `_mask` before the file extension:

```text
sample_001.png      -> sample_001_mask.png
sample_001.jpg      -> sample_001_mask.jpg
sample_001.bmp      -> sample_001_mask.bmp
```

Images and masks are paired by matching the filename stem before `_mask`.

Example loading with Pillow:

```python
from PIL import Image
import numpy as np

image = np.array(Image.open(
    "processed_ebhi/train/images/sample_001.png"
))
mask = np.array(Image.open(
    "processed_ebhi/train/masks/sample_001_mask.png"
))

print(image.shape, image.dtype)
print(mask.shape, mask.dtype)
```

Spatial dimensions are determined by the source files; the notebook does not resize either images or masks.

---

## Suggested `config.yaml`

After preprocessing, the notebook prints the following configuration:

```yaml
root_dir: processed_ebhi
mask_suffix: "_mask"
train: { images: train/images, masks: train/masks }
val:   { images: val/images,   masks: val/masks }
test:  { images: test/images,  masks: test/masks }
```

`mask_suffix` should match `NEW_MASK_SUFFIX` in the notebook.

---

## Running the Notebook

1. Place EBHI-SEG in the configured Kaggle dataset location.
2. Update `BASE_SRC` if your source files are stored elsewhere.
3. Run all cells in `preprocess.ipynb`.
4. Check the printed split summary:

```text
Train: ...
Val:   ...
Test:  ...
```

5. Use the generated `processed_ebhi` directory with the configuration above.

The output folder is recreated on every run, so do not store files that need to be preserved inside `DEST_DIR`.
---
EBHI can be download [here](https://www.kaggle.com/datasets/orvile/ebhi-seg-colorectal-cancer)