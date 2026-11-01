# GlaS Dataset Preprocessing

This preprocessing pipeline converts the Warwick-QU GlaS colon histology dataset into a clean, model-ready structure. The notebook:

1. Locates the source image and annotation files
2. Splits the training images into training and validation subsets
3. Copies images and matching masks into separate folders
4. Renames masks from `_anno` to `_mask`
5. Creates a combined `testAB` folder containing both official test subsets

The default output is organized as:

```text
processed_glas/
├── train/
│   ├── images/
│   └── masks/
├── val/
│   ├── images/
│   └── masks/
├── testA/
│   ├── images/
│   └── masks/
├── testB/
│   ├── images/
│   └── masks/
└── testAB/
    ├── images/
    └── masks/
```

---

## Dataset Source

The notebook is configured for the Kaggle dataset:

```python
SOURCE_DIR = "/kaggle/input/datasets/kangaonkaggle/glas-dataset/Warwick QU Dataset (Released 2016_07_08)"
DEST_DIR = "processed_glas"
```

Update `SOURCE_DIR` when running the notebook outside Kaggle. The source folder contains the original flat GlaS files, such as:

```text
Warwick QU Dataset (Released 2016_07_08)/
├── train_1.bmp
├── train_1_anno.bmp
├── train_2.bmp
├── train_2_anno.bmp
├── testA_1.bmp
├── testA_1_anno.bmp
├── testB_1.bmp
├── testB_1_anno.bmp
└── ...
```

Some releases may use uppercase `_Anno.bmp`, and the notebook checks both lowercase and uppercase variants.

---

## Dataset Split

The original 85 training images are shuffled and split deterministically:

```python
TRAIN_SPLIT = 0.8
SEED = 42
```

This produces approximately:

| Split | Images |
|---|---:|
| Train | 68 |
| Validation | 17 |
| Test A | 60 |
| Test B | 20 |
| Test A + Test B | 80 |

The split happens after:

```python
random.seed(SEED)
random.shuffle(all_train_imgs)
```

The first 80% becomes the training set and the remaining 20% becomes the validation set.

---

## Image and Mask Matching

Images and masks share a common sample identifier. Given an image such as:

```text
train_1.bmp
```

the notebook first looks for:

```text
train_1_anno.bmp
```

and then:

```text
train_1_Anno.bmp
```

If neither mask exists, the image is skipped with a warning.

The mask name is converted to the configured model-friendly suffix:

```python
NEW_MASK_SUFFIX = "_mask"
```

For example:

```text
train_1.bmp      -> train/images/train_1.bmp
train_1_anno.bmp -> train/masks/train_1_mask.bmp
```

---

## Preprocessing Steps

### 1. Create the output directories

The notebook creates `images` and `masks` directories for each supported split:

- `train`
- `val`
- `testA`
- `testB`
- `testAB`

If `processed_glas` already exists, it is removed before processing to prevent stale files from remaining in the output.

### 2. Discover source files

Files are collected with the original GlaS naming prefixes:

```python
all_train_imgs = sorted(glob.glob(os.path.join(SOURCE_DIR, "train_*.bmp")))
all_testA_imgs = sorted(glob.glob(os.path.join(SOURCE_DIR, "testA_*.bmp")))
all_testB_imgs = sorted(glob.glob(os.path.join(SOURCE_DIR, "testB_*.bmp")))
```

Mask files are excluded from the image lists using:

```python
if "_anno" not in f.lower()
```

### 3. Copy paired samples

For each image, `shutil.copy2()` copies:

- the original `.bmp` image to the split's `images` folder
- its matching annotation mask to the split's `masks` folder

Metadata is copied together with the files, but the image and mask contents are otherwise unchanged.

The notebook does not resize images, convert color channels, normalize pixels, or modify mask values.

### 4. Build the combined test folder

Test A and Test B images are written to both their individual folders and the combined folder:

```python
testAB/images/
testAB/masks/
```

This makes it possible to run evaluation over all 80 official test images while retaining the separate `testA` and `testB` splits.

---

## Output Format

Images use the original `*.bmp` file format. Masks also use `*.bmp`, with the stem followed by `_mask`.

The image and mask are paired by matching the filename stem:

```text
train_1.bmp       <-> train_1_mask.bmp
testA_10.bmp      <-> testA_10_mask.bmp
```

Example loading:

```python
from PIL import Image
import numpy as np

image = np.array(Image.open("processed_glas/train/images/train_1.bmp"))
mask = np.array(Image.open("processed_glas/train/masks/train_1_mask.bmp"))

print(image.shape, image.dtype)
print(mask.shape, mask.dtype)
```

Exact image dimensions vary by sample. Verify mask semantics in your training framework before use, because annotation mask values should be interpreted according to the original GlaS label definition.

---

## Suggested `config.yaml`

After preprocessing, the notebook prints a configuration compatible with the repository's image/mask split convention:

```yaml
root_dir: processed_glas
mask_suffix: "_mask"
train: { images: train/images, masks: train/masks }
val:   { images: val/images,   masks: val/masks }
test:  { images: testAB/images,  masks: testAB/masks }
```

For separate benchmark reporting, use:

```yaml
testA: { images: testA/images, masks: testA/masks }
testB: { images: testB/images, masks: testB/masks }
```

---

## Running the Notebook

1. Place the GlaS dataset in the Kaggle input location or update `SOURCE_DIR`.
2. Run all cells in `preprocess.ipynb`.
3. Confirm that the source scan reports the expected train, Test A, and Test B counts.
4. Use the generated `processed_glas` directory with the configuration shown above.

The output folder is recreated on every run, so do not place files you need to preserve inside `DEST_DIR`.

---
GlaS can be download [here](https://academictorrents.com/details/208814dd113c2b0a242e74e832ccac28fcff74e5)