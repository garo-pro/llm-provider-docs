> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

## Visualizing Open AI Embeddings in Atlas

In this example, we will upload food review embeddings to [Atlas](https://atlas.nomic.ai) to visualize the embeddings.

## What is Atlas?

[Atlas](https://atlas.nomic.ai) is a machine learning tool used to visualize massive datasets of embeddings in your web browser. Upload millions of embeddings to Atlas and interact with them in your web browser or jupyter notebook.

### 1. Login to Atlas.


```python
!pip install nomic
```

```python
import pandas as pd
import numpy as np
from ast import literal_eval

# Load the embeddings
datafile_path = "data/fine_food_reviews_with_embeddings_1k.csv"
df = pd.read_csv(datafile_path)

# Convert to a list of lists of floats
embeddings = np.array(df.embedding.apply(literal_eval).to_list())
df = df.drop('embedding', axis=1)
df = df.rename(columns={'Unnamed: 0': 'id'})
```

```python
import nomic
from nomic import atlas
nomic.login('7xDPkYXSYDc1_ErdTPIcoAR9RNd8YDlkS3nVNXcVoIMZ6') #demo account

data = df.to_dict('records')
project = atlas.map_embeddings(embeddings=embeddings, data=data,
                               id_field='id',
                               colorable_fields=['Score'])
map = project.maps[0]
```

### 2. Interact with your embeddings in Jupyter

```python
map
```

<h3>Project: meek-laborer</h3>
            

        <h4>Projection ID: 463f4614-7689-47e4-b55b-1da0cc679559</h4>
        <div class="actions">
            <div id="hide" class="action" onclick="destroy()">Hide embedded project</div>
            <div class="action" id="out">
                <a href="https://atlas.nomic.ai/map/fddc0e07-97c5-477c-827c-96bca44519aa/463f4614-7689-47e4-b55b-1da0cc679559" target="_blank">Explore on atlas.nomic.ai</a>
            </div>
        </div>
        
        <iframe class="iframe" id="iframe463f4614-7689-47e4-b55b-1da0cc679559" allow="clipboard-read; clipboard-write" src="https://atlas.nomic.ai/map/fddc0e07-97c5-477c-827c-96bca44519aa/463f4614-7689-47e4-b55b-1da0cc679559">
        </iframe>