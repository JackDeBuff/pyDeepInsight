# DeepInsight

This repository contains a python version of the image transformation procedure described in the paper [DeepInsight: A methodology to transform a non-image data to an image for convolution neural network architecture](https://doi.org/10.1038/s41598-019-47765-6).

## Installation

```
sudo pip install git+git://github.com/alok-ai-lab/DeepInsight.git#egg=DeepInsight
```

## Usage

The following is a walkthrough of standard usage of the ImageTransformer class

```python
from pyDeepInsight import ImageTransformer, LogScaler
from sklearn.model_selection import train_test_split
import pandas as pd
import numpy as np

from matplotlib import pyplot as plt
import matplotlib.ticker as ticker
import seaborn as sns
```
Load example TCGA data

```python
expr_file = r"./data/tcga.rnaseq_fpkm_uq.example.txt.gz"
expr = pd.read_csv(expr_file, sep="\t")
y = expr['project'].values
X = expr.iloc[:, 1:].values
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=23, stratify=y)
X_train.shape
```
    (480, 5000)

Normalize data to values between 0 and 1. The following normalization procedure is described in the [DeepInsight paper supplementary information](https://static-content.springer.com/esm/art%3A10.1038%2Fs41598-019-47765-6/MediaObjects/41598_2019_47765_MOESM1_ESM.pdf) as norm-2.

```python

ln = LogScaler()
X_train_norm = ln.fit_transform(X_train)
X_test_norm = ln.transform(X_test)

```
Initialize image transformer.

```python
it = ImageTransformer(feature_extractor='tsne', 
                      pixels=50, random_state=1701, 
                      n_jobs=-1)
```

Train image transformer on training data. Setting plot=True results in at a plot showing the reduced features (blue points), convex full (red), and minimum bounding rectagle (green) prior to rotation.

```python
plt.figure(figsize=(5, 5))
it.fit(X_train_norm, plot=True)
```

![png](./data/output_8_0.png)

The feature density matrix can be extracted from the trained transformer in order to view overall feature overlap.

```python
fdm = it.feature_density_matrix()
fdm[fdm == 0] = np.nan

plt.figure(figsize=(10, 7))

ax = sns.heatmap(fdm, cmap="viridis", linewidths=0.01, 
                 linecolor="lightgrey", square=True)
for _, spine in ax.spines.items():
    spine.set_visible(True)
```

![png](./data/output_9_0.png)

It is possible to update the pixel size without retraining.

```python
px_sizes = [25, (25, 50), 50, 100]

fig, ax = plt.subplots(1, len(px_sizes), figsize=(25, 7))
for ix, px in enumerate(px_sizes):
    it.pixels = px
    fdm = it.feature_density_matrix()
    fdm[fdm == 0] = np.nan
    cax = sns.heatmap(fdm, cmap="viridis", linewidth=0.01, 
                      linecolor="lightgrey", square=True, 
                      ax=ax[ix], cbar=False)
    cax.set_title('Dim {} x {}'.format(*it.pixels))
    for _, spine in cax.spines.items():
        spine.set_visible(True)

it.pixels = 50
```

![png](./data/output_9_5.png)

The trained transformer can then be used to transform sample data to image matricies.

```python
mat_train = it.transform(X_train_norm)
```

Fit and transform can be done in a single step.

```python
mat_train = it.fit_transform(X_train_norm)
```
The following are showing plots for the image matrices first four samples of the training set. 

```python
fig, ax = plt.subplots(1, 4, figsize=(25, 7))
for i in range(0,4):
    cax = sns.heatmap(mat_test[i], cmap='hot',
                      linewidth=0.01, linecolor='dimgrey',
                      square=True, ax=ax[i], cbar=False)
    cax.axis('off')
plt.tight_layout()
```

![png](./data/output_14_0.png)

Transforming the testing data is done the same as transforming the training data.

```python
mat_test = it.transform(X_test_norm)

fig, ax = plt.subplots(1, 4, figsize=(25, 7))
for i in range(0,4):
    cax = sns.heatmap(mat_test[i], cmap='hot',
                      linewidth=0.01, linecolor='dimgrey',
                      square=True, ax=ax[i], cbar=False)
    cax.axis('off')
plt.tight_layout()
```

![png](./data/output_17_0.png)

The images matrices can then be used as impute for the CNN model.