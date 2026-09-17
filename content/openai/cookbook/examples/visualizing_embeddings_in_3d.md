# Visualizing embeddings in 3D

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

The example uses [PCA](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html) to reduce the dimensionality of the embeddings from 1536 to 3. Then we can visualize the data points in a 3D plot. The small dataset `dbpedia_samples.jsonl` is curated by randomly sampling 200 samples from [DBpedia validation dataset](https://www.kaggle.com/danofer/dbpedia-classes?select=DBPEDIA_val.csv).

### 1. Load the dataset and query embeddings

```python
import pandas as pd
samples = pd.read_json("data/dbpedia_samples.jsonl", lines=True)
categories = sorted(samples["category"].unique())
print("Categories of DBpedia samples:", samples["category"].value_counts())
samples.head()
```

```text
Categories of DBpedia samples: Artist                    21
Film                      19
Plant                     19
OfficeHolder              18
Company                   17
NaturalPlace              16
Athlete                   16
Village                   12
WrittenWork               11
Building                  11
Album                     11
Animal                    11
EducationalInstitution    10
MeanOfTransportation       8
Name: category, dtype: int64
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>text</th>
      <th>category</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Morada Limited is a textile company based in ...</td>
      <td>Company</td>
    </tr>
    <tr>
      <th>1</th>
      <td>The Armenian Mirror-Spectator is a newspaper ...</td>
      <td>WrittenWork</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Mt. Kinka (金華山 Kinka-zan) also known as Kinka...</td>
      <td>NaturalPlace</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Planning the Play of a Bridge Hand is a book ...</td>
      <td>WrittenWork</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Wang Yuanping (born 8 December 1976) is a ret...</td>
      <td>Athlete</td>
    </tr>
  </tbody>
</table>
</div>

```python
from utils.embeddings_utils import get_embeddings
# NOTE: The following code will send a query of batch size 200 to /embeddings
matrix = get_embeddings(samples["text"].to_list(), model="text-embedding-3-small")
```

### 2. Reduce the embedding dimensionality

```python
from sklearn.decomposition import PCA
pca = PCA(n_components=3)
vis_dims = pca.fit_transform(matrix)
samples["embed_vis"] = vis_dims.tolist()
```

### 3. Plot the embeddings of lower dimensionality

```python
%matplotlib widget
import matplotlib.pyplot as plt
import numpy as np

fig = plt.figure(figsize=(10, 5))
ax = fig.add_subplot(projection='3d')
cmap = plt.get_cmap("tab20")

# Plot each sample category individually such that we can set label name.
for i, cat in enumerate(categories):
    sub_matrix = np.array(samples[samples["category"] == cat]["embed_vis"].to_list())
    x=sub_matrix[:, 0]
    y=sub_matrix[:, 1]
    z=sub_matrix[:, 2]
    colors = [cmap(i/len(categories))] * len(sub_matrix)
    ax.scatter(x, y, zs=z, zdir='z', c=colors, label=cat)

ax.set_xlabel('x')
ax.set_ylabel('y')
ax.set_zlabel('z')
ax.legend(bbox_to_anchor=(1.1, 1))
```

```text
<matplotlib.legend.Legend at 0x1622180a0>
```

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/visualizing_embeddings_in_3d/cell-8-output-1.png)