# MoNuSeg Dataset Preprocessing

This preprocessing pipeline converts the original MoNuSeg dataset into a clean training/testing structure that is easier to use for nuclei segmentation experiments. The workflow in the notebook does three main things:

1. Finds the original image and annotation directories in the source dataset
2. Copies the files into a prepared folder structure
3. Converts the TIFF images and XML annotations into a model-friendly format

The final output is organized as:

```text
monuseg_converted/
├── training/
│   ├── images/
│   │   ├── image_001.png
│   │   ├── image_002.png
│   │   └── ...
│   └── labels/
│       ├── image_001.npy
│       ├── image_002.npy
│       └── ...
├── testing/
│   ├── images/
│   │   ├── test_image_001.png
│   │   ├── test_image_002.png
│   │   └── ...
│   └── labels/
│       ├── test_image_001.npy
│       ├── test_image_002.npy
│       └── ...
└── ...
```

---

## Raw Dataset Structure

The original MoNuSeg dataset contains microscopy images and XML annotations. The notebook first searches the input tree for directories containing either `.tif` images or `.xml` annotations.

```python
from pathlib import Path
import shutil

root = Path("/kaggle/input")
for p in root.rglob("*"):
    if p.is_dir() and ("MoNuSeg" in p.name or "Images" in p.name or "Annotations" in p.name):
        print(p)
```

The code looks through the filesystem recursively and collects all directories that contain image files or XML annotation files.

It then creates the expected output structure:

```python
dst = Path("/kaggle/working/monuseg_prepared")
(dst / "training" / "images").mkdir(parents=True, exist_ok=True)
(dst / "training" / "labels").mkdir(parents=True, exist_ok=True)
(dst / "testing" / "images").mkdir(parents=True, exist_ok=True)
(dst / "testing" / "labels").mkdir(parents=True, exist_ok=True)
```

The copy step separates files by naming pattern:

```python
for d in image_dirs:
    for f in d.glob("*.tif"):
        if "Test" in str(d) or "Test" in f.name:
            shutil.copy(f, dst / "testing" / "images" / f.name)
        else:
            shutil.copy(f, dst / "training" / "images" / f.name)
```

and similarly for XML label files.

This means the dataset is organized into train/test splits based on whether the folder or filename includes `Test`.

---

## Data Types in the Original Source

The original MoNuSeg files use the following data types:

- Images: TIFF (`.tif`) files
- Labels: XML annotation files (`.xml`)
- The XML files store polygon annotations for individual nuclei instances

The conversion script does not use the XML directly as a mask. Instead, it parses the polygon vertices and reconstructs a binary mask from them.

---

## Conversion Pipeline

The main conversion function is:

```python
def convert_monuseg(input_path: Union[Path, str], output_path: Union[Path, str]) -> None:
```

This function does the following:

### 1. Prepare output folders

```python
parts = ["testing", "training"]
for part in parts:
    input_path_part = input_path / part
    output_path_part = output_path / part
    output_path_part.mkdir(exist_ok=True, parents=True)
    (output_path_part / "images").mkdir(exist_ok=True, parents=True)
    (output_path_part / "labels").mkdir(exist_ok=True, parents=True)
```

The output is split into two folders: `training` and `testing`, each containing `images` and `labels` directories.

### 2. Convert all images

```python
images = [f for f in sorted((input_path_part / "images").glob("*.tif"))]
for img_path in images:
    loaded_image = Image.open(img_path)
    resized = loaded_image.resize((1000, 1000), resample=Image.Resampling.LANCZOS)
    new_img_path = output_path_part / "images" / f"{img_path.stem}.png"
    resized.save(new_img_path)
```

What happens here:

- each TIFF image is opened with PIL
- it is resized to `1000 x 1000`
- it is saved as a PNG file
- the image name is kept as the original stem, so the file names remain consistent with the source annotation names

#### Image output type

- File format: `.png`
- Image array type after loading: `PIL.Image.Image`
- Final spatial size: `(1000, 1000)`
- Channels: grayscale image, usually single-channel, saved visually as PNG

#### Typical image representation

When loaded in Python:

```python
from PIL import Image
img = Image.open("image_001.png")
arr = np.array(img)
print(arr.shape, arr.dtype)
```

Expected result:

```python
(1000, 1000) uint8
```

The exact color mode depends on the source image, but for MoNuSeg microscopy images this is typically a grayscale or single-channel image.

---

### 3. Convert XML annotations into segmentation masks

The code reads each XML annotation file:

```python
annotations = [f for f in sorted((input_path_part / "labels").glob("*.xml"))]
for annot_path in annotations:
    binary_mask = np.zeros((1000, 1000), dtype=np.int32)

    tree = ET.parse(annot_path)
    root = tree.getroot()
    child = root[0]
```

The XML structure contains polygon regions representing each nucleus. The code parses the annotation tree and then extracts polygon coordinates:

```python
for x in child:
    r = x.tag
    if r == "Regions":
        element_idx = 1
        for y in x:
            y_tag = y.tag

            if y_tag == "Region":
                regions = []
                vertices = y[1]
                coords = np.zeros((len(vertices), 2))
                for i, vertex in enumerate(vertices):
                    coords[i][0] = vertex.attrib["X"]
                    coords[i][1] = vertex.attrib["Y"]
                regions.append(coords)
                vertex_row_coords = regions[0][:, 0]
                vertex_col_coords = regions[0][:, 1]
                fill_row_coords, fill_col_coords = draw.polygon(
                    vertex_col_coords, vertex_row_coords, binary_mask.shape
                )
                binary_mask[fill_row_coords, fill_col_coords] = element_idx

                element_idx = element_idx + 1
```

#### What this does

- each nucleus region is defined by polygon vertices
- those vertices are converted into image coordinates
- `skimage.draw.polygon` fills the polygon region inside the mask
- each nucleus is assigned a different integer ID based on `element_idx`

This means the resulting label mask is an instance segmentation mask, not a binary foreground/background mask.

#### Important label semantics

- `0` means background
- `1`, `2`, `3`, ... indicate different nuclei instances
- each nucleus gets its own integer region ID

Example:

```python
mask = np.load("image_001.npy")
print(np.unique(mask))
```

Possible output:

```python
[0 1 2 3 4 5]
```

This means the image contains 5 identified nuclei and background pixels are zero.

---

### 4. Resize the instance mask to 1000 x 1000

```python
inst_image = Image.fromarray(binary_mask)
resized_mask = np.array(
    inst_image.resize((1000, 1000), resample=Image.Resampling.NEAREST)
)
new_mask_path = output_path_part / "labels" / f"{annot_path.stem}.npy"
np.save(new_mask_path, resized_mask)
```

Here, the mask is converted into a PIL image and resized with nearest-neighbor interpolation to avoid mixing class IDs.

#### Mask output type

- File format: `.npy`
- Saved type: `numpy.ndarray`
- Shape: `(1000, 1000)`
- Dtype: `np.int32` before saving
- After `np.save`, the file contains a NumPy array with integer labels

This is an instance segmentation label map, meaning each nucleus is represented by a distinct integer.

---

## Final Output Format

The final converted dataset contains:

### Training images

- Folder: `training/images`
- Files: `.png`
- Example: `tumor_001.png`
- Shape when loaded: `(1000, 1000)`
- dtype: usually `uint8`

### Training labels

- Folder: `training/labels`
- Files: `.npy`
- Example: `tumor_001.npy`
- Shape when loaded: `(1000, 1000)`
- dtype: integer, typically `int32` or `int64` depending on the saved array

### Testing images

- Folder: `testing/images`
- Files: `.png`
- Shape: `(1000, 1000)`
- dtype: `uint8`

### Testing labels

- Folder: `testing/labels`
- Files: `.npy`
- Shape: `(1000, 1000)`
- dtype: integer labels representing nucleus instances

---

## How to Use the Output Data

### Load an image

```python
from PIL import Image
import numpy as np

img = np.array(Image.open("./monuseg_converted/training/images/image_001.png"))
print(img.shape, img.dtype)
```

Expected:

```python
(1000, 1000) uint8
```

### Load a mask

```python
mask = np.load("./monuseg_converted/training/labels/image_001.npy")
print(mask.shape, mask.dtype)
print(np.unique(mask))
```

Expected:

```python
(1000, 1000) int32
[0 1 2 3 ...]
```

This means the mask is a segmentation map where:

- `0` = background
- `1`, `2`, `3`, ... = different nuclei instances

---

## Important Notes for Training

This dataset is converted for instance segmentation, not semantic segmentation.

That means:

- the model is expected to detect individual nuclei instances
- the mask does not just say “nucleus vs background”
- each nucleus instance is assigned a unique integer region label

A common training setup would do one of the following:

1. Treat the problem as instance segmentation and predict one mask per nucleus
2. Convert the instance mask into a binary nucleus mask by setting all nonzero values to `1`
3. Use connected-component grouping to separate nuclei for evaluation

### Example binary conversion

```python
mask = np.load("./monuseg_converted/training/labels/image_001.npy")
binary_mask = (mask > 0).astype(np.uint8)
print(binary_mask.shape, binary_mask.dtype)
print(np.unique(binary_mask))
```

This produces:

```python
(1000, 1000) uint8
[0 1]
```

This is useful if you want a simple nucleus-vs-background segmentation map instead of instance labels.

---

## Summary of Data Types

### Image files

- Type: `.png`
- Loaded as: `numpy.ndarray`
- Shape: `(1000, 1000)`
- Dtype: usually `uint8` for grayscale microscopy images

### Label files

- Type: `.npy`
- Loaded as: `numpy.ndarray`
- Shape: `(1000, 1000)`
- Dtype: integer mask values
- Semantics: instance IDs for nuclei; background is `0`

---

## Final Takeaway

The dataset is converted into a clean structure where:

- images are resized to `1000 x 1000` and saved as PNG
- annotation XML polygons are transformed into instance masks
- each nucleus gets its own integer label in the mask
- training and testing folders are created separately
- the output is designed for segmentation pipelines and easy DataLoader usage

When you revisit the data later, the key thing to remember is:

- image arrays are grayscale microscopy images
- label arrays are instance-level nuclei masks
- a pixel value of `0` is background
- a positive nonzero value indicates a nucleus instance ID

This is the most important behavior of the MoNuSeg preprocessing pipeline. 