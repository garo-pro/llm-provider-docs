# Get embeddings from dataset

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook gives an example on how to get embeddings from a large dataset.


## 1. Load the dataset

The dataset used in this example is [fine-food reviews](https://www.kaggle.com/snap/amazon-fine-food-reviews) from Amazon. The dataset contains a total of 568,454 food reviews Amazon users left up to October 2012. We will use a subset of this dataset, consisting of 1,000 most recent reviews for illustration purposes. The reviews are in English and tend to be positive or negative. Each review has a ProductId, UserId, Score, review title (Summary) and review body (Text).

We will combine the review summary and review text into a single combined text. The model will encode this combined text and it will output a single vector embedding.

To run this notebook, you will need to install: pandas, openai, transformers, plotly, matplotlib, scikit-learn, torch (transformer dep), torchvision, and scipy.

```python
import pandas as pd
import tiktoken

from utils.embeddings_utils import get_embedding
```

```python
embedding_model = "text-embedding-3-small"
embedding_encoding = "cl100k_base"
max_tokens = 8000  # the maximum for text-embedding-3-small is 8191
```

```python
# load & inspect dataset
input_datapath = "data/fine_food_reviews_1k.csv"  # to save space, we provide a pre-filtered dataset
df = pd.read_csv(input_datapath, index_col=0)
df = df[["Time", "ProductId", "UserId", "Score", "Summary", "Text"]]
df = df.dropna()
df["combined"] = (
    "Title: " + df.Summary.str.strip() + "; Content: " + df.Text.str.strip()
)
df.head(2)
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Time</th>
      <th>ProductId</th>
      <th>UserId</th>
      <th>Score</th>
      <th>Summary</th>
      <th>Text</th>
      <th>combined</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1351123200</td>
      <td>B003XPF9BO</td>
      <td>A3R7JR3FMEBXQB</td>
      <td>5</td>
      <td>where does one  start...and stop... with a tre...</td>
      <td>Wanted to save some to bring to my Chicago fam...</td>
      <td>Title: where does one  start...and stop... wit...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1351123200</td>
      <td>B003JK537S</td>
      <td>A3JBPC3WFUT5ZP</td>
      <td>1</td>
      <td>Arrived in pieces</td>
      <td>Not pleased at all. When I opened the box, mos...</td>
      <td>Title: Arrived in pieces; Content: Not pleased...</td>
    </tr>
  </tbody>
</table>
</div>

```python
# subsample to 1k most recent reviews and remove samples that are too long
top_n = 1000
df = df.sort_values("Time").tail(top_n * 2)  # first cut to first 2k entries, assuming less than half will be filtered out
df.drop("Time", axis=1, inplace=True)

encoding = tiktoken.get_encoding(embedding_encoding)

# omit reviews that are too long to embed
df["n_tokens"] = df.combined.apply(lambda x: len(encoding.encode(x)))
df = df[df.n_tokens <= max_tokens].tail(top_n)
len(df)
```

```text
1000
```

## 2. Get embeddings and save them for future reuse

```python
# Ensure you have your API key set in your environment per the README: https://github.com/openai/openai-python#usage

# This may take a few minutes
df["embedding"] = df.combined.apply(lambda x: get_embedding(x, model=embedding_model))
df.to_csv("data/fine_food_reviews_with_embeddings_1k.csv")
```

```python
a = get_embedding("hi", model=embedding_model)
```