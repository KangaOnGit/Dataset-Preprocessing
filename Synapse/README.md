## The Synapse Abdomen Dataset was preprocessed using TransUNet's Methods: [TransUNet](https://github.com/Beckschen/TransUNet/tree/main/datasets)

Please note that the preprocessed data retains the original 13 Labels of the Abdomen Dataset. Please remap the labels yourself.
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
