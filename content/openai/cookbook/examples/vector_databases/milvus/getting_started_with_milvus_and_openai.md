# Getting Started with Milvus and OpenAI

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

### Finding your next book

In this notebook we will be going over generating embeddings of book descriptions with OpenAI and using those embeddings within Milvus to find relevant books. The dataset in this example is sourced from HuggingFace datasets, and contains a little over 1 million title-description pairs.

Lets begin by first downloading the required libraries for this notebook:
- `openai` is used for communicating with the OpenAI embedding service
- `pymilvus` is used for communicating with the Milvus server
- `datasets` is used for downloading the dataset
- `tqdm` is used for the progress bars


```python
! pip install openai pymilvus datasets tqdm
```

```text
Looking in indexes: https://pypi.org/simple, https://pypi.ngc.nvidia.com
Requirement already satisfied: openai in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (0.27.2)
Requirement already satisfied: pymilvus in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (2.2.2)
Requirement already satisfied: datasets in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (2.10.1)
Requirement already satisfied: tqdm in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (4.64.1)
Requirement already satisfied: aiohttp in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from openai) (3.8.4)
Requirement already satisfied: requests>=2.20 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from openai) (2.28.2)
Requirement already satisfied: pandas>=1.2.4 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pymilvus) (1.5.3)
Requirement already satisfied: ujson<=5.4.0,>=2.0.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pymilvus) (5.1.0)
Requirement already satisfied: mmh3<=3.0.0,>=2.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pymilvus) (3.0.0)
Requirement already satisfied: grpcio<=1.48.0,>=1.47.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pymilvus) (1.47.2)
Requirement already satisfied: grpcio-tools<=1.48.0,>=1.47.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pymilvus) (1.47.2)
Requirement already satisfied: huggingface-hub<1.0.0,>=0.2.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (0.12.1)
Requirement already satisfied: dill<0.3.7,>=0.3.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (0.3.6)
Requirement already satisfied: xxhash in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (3.2.0)
Requirement already satisfied: pyyaml>=5.1 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (5.4.1)
Requirement already satisfied: fsspec[http]>=2021.11.1 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (2023.1.0)
Requirement already satisfied: packaging in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (23.0)
Requirement already satisfied: numpy>=1.17 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (1.23.5)
Requirement already satisfied: multiprocess in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (0.70.14)
Requirement already satisfied: pyarrow>=6.0.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (10.0.1)
Requirement already satisfied: responses<0.19 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from datasets) (0.18.0)
Requirement already satisfied: multidict<7.0,>=4.5 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (6.0.4)
Requirement already satisfied: frozenlist>=1.1.1 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (1.3.3)
Requirement already satisfied: async-timeout<5.0,>=4.0.0a3 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (4.0.2)
Requirement already satisfied: yarl<2.0,>=1.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (1.8.2)
Requirement already satisfied: aiosignal>=1.1.2 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (1.3.1)
Requirement already satisfied: charset-normalizer<4.0,>=2.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (3.0.1)
Requirement already satisfied: attrs>=17.3.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from aiohttp->openai) (22.2.0)
Requirement already satisfied: six>=1.5.2 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from grpcio<=1.48.0,>=1.47.0->pymilvus) (1.16.0)
Requirement already satisfied: protobuf<4.0dev,>=3.12.0 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from grpcio-tools<=1.48.0,>=1.47.0->pymilvus) (3.20.1)
Requirement already satisfied: setuptools in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from grpcio-tools<=1.48.0,>=1.47.0->pymilvus) (65.6.3)
Requirement already satisfied: filelock in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from huggingface-hub<1.0.0,>=0.2.0->datasets) (3.9.0)
Requirement already satisfied: typing-extensions>=3.7.4.3 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from huggingface-hub<1.0.0,>=0.2.0->datasets) (4.5.0)
Requirement already satisfied: python-dateutil>=2.8.1 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pandas>=1.2.4->pymilvus) (2.8.2)
Requirement already satisfied: pytz>=2020.1 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from pandas>=1.2.4->pymilvus) (2022.7.1)
Requirement already satisfied: urllib3<1.27,>=1.21.1 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from requests>=2.20->openai) (1.26.14)
Requirement already satisfied: idna<4,>=2.5 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from requests>=2.20->openai) (3.4)
Requirement already satisfied: certifi>=2017.4.17 in /Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages (from requests>=2.20->openai) (2022.12.7)
```

With the required packages installed we can get started. Lets begin by launching the Milvus service. The file being run is the `docker-compose.yaml` found in the folder of this file. This command launches a Milvus standalone instance which we will use for this test.  

```python
! docker compose up -d
```

```text
[1A[1B[0G[?25l[+] Running 0/0
[37m ⠋ Network milvus  Creating                                                0.1s
[0m[?25h[1A[1A[0G[?25l[34m[+] Running 1/1[0m
[34m ⠿ Network milvus          Created                                         0.1s
[0m[37m ⠋ Container milvus-minio  Creating                                        0.1s
[0m[37m ⠋ Container milvus-etcd   Creating                                        0.1s
[0m[?25h[1A[1A[1A[1A[0G[?25l[+] Running 1/3
[34m ⠿ Network milvus          Created                                         0.1s
[0m[37m ⠙ Container milvus-minio  Creating                                        0.2s
[0m[37m ⠙ Container milvus-etcd   Creating                                        0.2s
[0m[?25h[1A[1A[1A[1A[0G[?25l[+] Running 1/3
[34m ⠿ Network milvus          Created                                         0.1s
[0m[37m ⠹ Container milvus-minio  Creating                                        0.3s
[0m[37m ⠹ Container milvus-etcd   Creating                                        0.3s
[0m[?25h[1A[1A[1A[1A[0G[?25l[34m[+] Running 3/3[0m
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Created                                    0.3s
[0m[34m ⠿ Container milvus-etcd        Created                                    0.3s
[0m[37m ⠋ Container milvus-standalone  Creating                                   0.1s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Created                                    0.3s
[0m[34m ⠿ Container milvus-etcd        Created                                    0.3s
[0m[37m ⠙ Container milvus-standalone  Creating                                   0.2s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[34m[+] Running 4/4[0m
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Created                                    0.3s
[0m[34m ⠿ Container milvus-etcd        Created                                    0.3s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   0.7s
[0m[37m ⠿ Container milvus-etcd        Starting                                   0.7s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   0.8s
[0m[37m ⠿ Container milvus-etcd        Starting                                   0.8s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   0.9s
[0m[37m ⠿ Container milvus-etcd        Starting                                   0.9s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.0s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.0s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.1s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.1s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.2s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.2s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.3s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.3s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.4s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.4s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.5s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.5s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.6s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.6s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 2/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.7s
[0m[37m ⠿ Container milvus-etcd        Starting                                   1.7s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[37m ⠿ Container milvus-minio       Starting                                   1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[34m ⠿ Container milvus-standalone  Created                                    0.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   1.6s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   1.7s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   1.8s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   1.9s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.0s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.1s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.2s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.3s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.4s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.5s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[+] Running 3/4
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[37m ⠿ Container milvus-standalone  Starting                                   2.6s
[0m[?25h[1A[1A[1A[1A[1A[0G[?25l[34m[+] Running 4/4[0m
[34m ⠿ Network milvus               Created                                    0.1s
[0m[34m ⠿ Container milvus-minio       Started                                    1.8s
[0m[34m ⠿ Container milvus-etcd        Started                                    1.7s
[0m[34m ⠿ Container milvus-standalone  Started                                    2.6s
[0m[?25h
```

With Milvus running we can setup our global variables:
- HOST: The Milvus host address
- PORT: The Milvus port number
- COLLECTION_NAME: What to name the collection within Milvus
- DIMENSION: The dimension of the embeddings
- OPENAI_ENGINE: Which embedding model to use
- openai.api_key: Your OpenAI account key
- INDEX_PARAM: The index settings to use for the collection
- QUERY_PARAM: The search parameters to use
- BATCH_SIZE: How many texts to embed and insert at once

```python
import openai

HOST = 'localhost'
PORT = 19530
COLLECTION_NAME = 'book_search'
DIMENSION = 1536
OPENAI_ENGINE = 'text-embedding-3-small'
openai.api_key = 'sk-your_key'

INDEX_PARAM = {
    'metric_type':'L2',
    'index_type':"HNSW",
    'params':{'M': 8, 'efConstruction': 64}
}

QUERY_PARAM = {
    "metric_type": "L2",
    "params": {"ef": 64},
}

BATCH_SIZE = 1000
```

## Milvus
This segment deals with Milvus and setting up the database for this use case. Within Milvus we need to setup a collection and index the collection. 

```python
from pymilvus import connections, utility, FieldSchema, Collection, CollectionSchema, DataType

# Connect to Milvus Database
connections.connect(host=HOST, port=PORT)
```

```python
# Remove collection if it already exists
if utility.has_collection(COLLECTION_NAME):
    utility.drop_collection(COLLECTION_NAME)
```

```python
# Create collection which includes the id, title, and embedding.
fields = [
    FieldSchema(name='id', dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name='title', dtype=DataType.VARCHAR, max_length=64000),
    FieldSchema(name='description', dtype=DataType.VARCHAR, max_length=64000),
    FieldSchema(name='embedding', dtype=DataType.FLOAT_VECTOR, dim=DIMENSION)
]
schema = CollectionSchema(fields=fields)
collection = Collection(name=COLLECTION_NAME, schema=schema)
```

```python
# Create the index on the collection and load it.
collection.create_index(field_name="embedding", index_params=INDEX_PARAM)
collection.load()
```

## Dataset
With Milvus up and running we can begin grabbing our data. Hugging Face Datasets is a hub that holds many different user datasets, and for this example we are using Skelebor's book dataset. This dataset contains title-description pairs for over 1 million books. We are going to embed each description and store it within Milvus along with its title. 

```python
import datasets

# Download the dataset and only use the `train` portion (file is around 800Mb)
dataset = datasets.load_dataset('Skelebor/book_titles_and_descriptions_en_clean', split='train')
```

```text
/Users/filiphaltmayer/miniconda3/envs/haystack/lib/python3.9/site-packages/tqdm/auto.py:22: TqdmWarning: IProgress not found. Please update jupyter and ipywidgets. See https://ipywidgets.readthedocs.io/en/stable/user_install.html
  from .autonotebook import tqdm as notebook_tqdm
Found cached dataset parquet (/Users/filiphaltmayer/.cache/huggingface/datasets/Skelebor___parquet/Skelebor--book_titles_and_descriptions_en_clean-3596935b1d8a7747/0.0.0/2a3b91fbd88a2c90d1dbbb32b460cf621d31bd5b05b934492fdef7d8d6f236ec)
```

## Insert the Data
Now that we have our data on our machine we can begin embedding it and inserting it into Milvus. The embedding function takes in text and returns the embeddings in a list format. 

```python
# Simple function that converts the texts to embeddings
def embed(texts):
    embeddings = openai.Embedding.create(
        input=texts,
        engine=OPENAI_ENGINE
    )
    return [x['embedding'] for x in embeddings['data']]
```

This next step inserts the data. Because there are so many data points, you can stop the insertion cell early if you want to test the workflow immediately. Using fewer data points will probably reduce the accuracy of the results, but it should still be sufficient for a test. 

```python
from tqdm import tqdm

data = [
    [], # title
    [], # description
]

# Embed and insert in batches
for i in tqdm(range(0, len(dataset))):
    data[0].append(dataset[i]['title'])
    data[1].append(dataset[i]['description'])
    if len(data[0]) % BATCH_SIZE == 0:
        data.append(embed(data[1]))
        collection.insert(data)
        data = [[],[]]

# Embed and insert the remainder 
if len(data[0]) != 0:
    data.append(embed(data[1]))
    collection.insert(data)
    data = [[],[]]
```

```text
  0%|          | 1999/1032335 [00:06<57:22, 299.31it/s]
```

```text
KeyboardInterrupt
[0;31m---------------------------------------------------------------------------[0m[0;31mKeyboardInterrupt[0m                         Traceback (most recent call last)Cell [0;32mIn[18], line 13[0m
[1;32m     11[0m data[[39m1[39m][39m.[39mappend(dataset[i][[39m'[39m[39mdescription[39m[39m'[39m])
[1;32m     12[0m [39mif[39;00m [39mlen[39m(data[[39m0[39m]) [39m%[39m BATCH_SIZE [39m==[39m [39m0[39m:
[0;32m---> 13[0m     data[39m.[39mappend(embed(data[[39m1[39;49m]))
[1;32m     14[0m     collection[39m.[39minsert(data)
[1;32m     15[0m     data [39m=[39m [[],[]]
Cell [0;32mIn[17], line 3[0m, in [0;36membed[0;34m(texts)[0m
[1;32m      2[0m [39mdef[39;00m [39membed[39m(texts):
[0;32m----> 3[0m     embeddings [39m=[39m openai[39m.[39;49mEmbedding[39m.[39;49mcreate(
[1;32m      4[0m         [39minput[39;49m[39m=[39;49mtexts,
[1;32m      5[0m         engine[39m=[39;49mOPENAI_ENGINE
[1;32m      6[0m     )
[1;32m      7[0m     [39mreturn[39;00m [x[[39m'[39m[39membedding[39m[39m'[39m] [39mfor[39;00m x [39min[39;00m embeddings[[39m'[39m[39mdata[39m[39m'[39m]]
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/openai/api_resources/embedding.py:33[0m, in [0;36mEmbedding.create[0;34m(cls, *args, **kwargs)[0m
[1;32m     31[0m [39mwhile[39;00m [39mTrue[39;00m:
[1;32m     32[0m     [39mtry[39;00m:
[0;32m---> 33[0m         response [39m=[39m [39msuper[39;49m()[39m.[39;49mcreate([39m*[39;49margs, [39m*[39;49m[39m*[39;49mkwargs)
[1;32m     35[0m         [39m# If a user specifies base64, we'll just return the encoded string.[39;00m
[1;32m     36[0m         [39m# This is only for the default case.[39;00m
[1;32m     37[0m         [39mif[39;00m [39mnot[39;00m user_provided_encoding_format:
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/openai/api_resources/abstract/engine_api_resource.py:153[0m, in [0;36mEngineAPIResource.create[0;34m(cls, api_key, api_base, api_type, request_id, api_version, organization, **params)[0m
[1;32m    127[0m [39m@classmethod[39m
[1;32m    128[0m [39mdef[39;00m [39mcreate[39m(
[1;32m    129[0m     [39mcls[39m,
[0;32m   (...)[0m
[1;32m    136[0m     [39m*[39m[39m*[39mparams,
[1;32m    137[0m ):
[1;32m    138[0m     (
[1;32m    139[0m         deployment_id,
[1;32m    140[0m         engine,
[0;32m   (...)[0m
[1;32m    150[0m         api_key, api_base, api_type, api_version, organization, [39m*[39m[39m*[39mparams
[1;32m    151[0m     )
[0;32m--> 153[0m     response, _, api_key [39m=[39m requestor[39m.[39;49mrequest(
[1;32m    154[0m         [39m"[39;49m[39mpost[39;49m[39m"[39;49m,
[1;32m    155[0m         url,
[1;32m    156[0m         params[39m=[39;49mparams,
[1;32m    157[0m         headers[39m=[39;49mheaders,
[1;32m    158[0m         stream[39m=[39;49mstream,
[1;32m    159[0m         request_id[39m=[39;49mrequest_id,
[1;32m    160[0m         request_timeout[39m=[39;49mrequest_timeout,
[1;32m    161[0m     )
[1;32m    163[0m     [39mif[39;00m stream:
[1;32m    164[0m         [39m# must be an iterator[39;00m
[1;32m    165[0m         [39massert[39;00m [39mnot[39;00m [39misinstance[39m(response, OpenAIResponse)
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/openai/api_requestor.py:216[0m, in [0;36mAPIRequestor.request[0;34m(self, method, url, params, headers, files, stream, request_id, request_timeout)[0m
[1;32m    205[0m [39mdef[39;00m [39mrequest[39m(
[1;32m    206[0m     [39mself[39m,
[1;32m    207[0m     method,
[0;32m   (...)[0m
[1;32m    214[0m     request_timeout: Optional[Union[[39mfloat[39m, Tuple[[39mfloat[39m, [39mfloat[39m]]] [39m=[39m [39mNone[39;00m,
[1;32m    215[0m ) [39m-[39m[39m>[39m Tuple[Union[OpenAIResponse, Iterator[OpenAIResponse]], [39mbool[39m, [39mstr[39m]:
[0;32m--> 216[0m     result [39m=[39m [39mself[39;49m[39m.[39;49mrequest_raw(
[1;32m    217[0m         method[39m.[39;49mlower(),
[1;32m    218[0m         url,
[1;32m    219[0m         params[39m=[39;49mparams,
[1;32m    220[0m         supplied_headers[39m=[39;49mheaders,
[1;32m    221[0m         files[39m=[39;49mfiles,
[1;32m    222[0m         stream[39m=[39;49mstream,
[1;32m    223[0m         request_id[39m=[39;49mrequest_id,
[1;32m    224[0m         request_timeout[39m=[39;49mrequest_timeout,
[1;32m    225[0m     )
[1;32m    226[0m     resp, got_stream [39m=[39m [39mself[39m[39m.[39m_interpret_response(result, stream)
[1;32m    227[0m     [39mreturn[39;00m resp, got_stream, [39mself[39m[39m.[39mapi_key
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/openai/api_requestor.py:516[0m, in [0;36mAPIRequestor.request_raw[0;34m(self, method, url, params, supplied_headers, files, stream, request_id, request_timeout)[0m
[1;32m    514[0m     _thread_context[39m.[39msession [39m=[39m _make_session()
[1;32m    515[0m [39mtry[39;00m:
[0;32m--> 516[0m     result [39m=[39m _thread_context[39m.[39;49msession[39m.[39;49mrequest(
[1;32m    517[0m         method,
[1;32m    518[0m         abs_url,
[1;32m    519[0m         headers[39m=[39;49mheaders,
[1;32m    520[0m         data[39m=[39;49mdata,
[1;32m    521[0m         files[39m=[39;49mfiles,
[1;32m    522[0m         stream[39m=[39;49mstream,
[1;32m    523[0m         timeout[39m=[39;49mrequest_timeout [39mif[39;49;00m request_timeout [39melse[39;49;00m TIMEOUT_SECS,
[1;32m    524[0m     )
[1;32m    525[0m [39mexcept[39;00m requests[39m.[39mexceptions[39m.[39mTimeout [39mas[39;00m e:
[1;32m    526[0m     [39mraise[39;00m error[39m.[39mTimeout([39m"[39m[39mRequest timed out: [39m[39m{}[39;00m[39m"[39m[39m.[39mformat(e)) [39mfrom[39;00m [39me[39;00m
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/requests/sessions.py:587[0m, in [0;36mSession.request[0;34m(self, method, url, params, data, headers, cookies, files, auth, timeout, allow_redirects, proxies, hooks, stream, verify, cert, json)[0m
[1;32m    582[0m send_kwargs [39m=[39m {
[1;32m    583[0m     [39m"[39m[39mtimeout[39m[39m"[39m: timeout,
[1;32m    584[0m     [39m"[39m[39mallow_redirects[39m[39m"[39m: allow_redirects,
[1;32m    585[0m }
[1;32m    586[0m send_kwargs[39m.[39mupdate(settings)
[0;32m--> 587[0m resp [39m=[39m [39mself[39;49m[39m.[39;49msend(prep, [39m*[39;49m[39m*[39;49msend_kwargs)
[1;32m    589[0m [39mreturn[39;00m resp
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/requests/sessions.py:701[0m, in [0;36mSession.send[0;34m(self, request, **kwargs)[0m
[1;32m    698[0m start [39m=[39m preferred_clock()
[1;32m    700[0m [39m# Send the request[39;00m
[0;32m--> 701[0m r [39m=[39m adapter[39m.[39;49msend(request, [39m*[39;49m[39m*[39;49mkwargs)
[1;32m    703[0m [39m# Total elapsed time of the request (approximately)[39;00m
[1;32m    704[0m elapsed [39m=[39m preferred_clock() [39m-[39m start
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/requests/adapters.py:489[0m, in [0;36mHTTPAdapter.send[0;34m(self, request, stream, timeout, verify, cert, proxies)[0m
[1;32m    487[0m [39mtry[39;00m:
[1;32m    488[0m     [39mif[39;00m [39mnot[39;00m chunked:
[0;32m--> 489[0m         resp [39m=[39m conn[39m.[39;49murlopen(
[1;32m    490[0m             method[39m=[39;49mrequest[39m.[39;49mmethod,
[1;32m    491[0m             url[39m=[39;49murl,
[1;32m    492[0m             body[39m=[39;49mrequest[39m.[39;49mbody,
[1;32m    493[0m             headers[39m=[39;49mrequest[39m.[39;49mheaders,
[1;32m    494[0m             redirect[39m=[39;49m[39mFalse[39;49;00m,
[1;32m    495[0m             assert_same_host[39m=[39;49m[39mFalse[39;49;00m,
[1;32m    496[0m             preload_content[39m=[39;49m[39mFalse[39;49;00m,
[1;32m    497[0m             decode_content[39m=[39;49m[39mFalse[39;49;00m,
[1;32m    498[0m             retries[39m=[39;49m[39mself[39;49m[39m.[39;49mmax_retries,
[1;32m    499[0m             timeout[39m=[39;49mtimeout,
[1;32m    500[0m         )
[1;32m    502[0m     [39m# Send the request.[39;00m
[1;32m    503[0m     [39melse[39;00m:
[1;32m    504[0m         [39mif[39;00m [39mhasattr[39m(conn, [39m"[39m[39mproxy_pool[39m[39m"[39m):
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/urllib3/connectionpool.py:703[0m, in [0;36mHTTPConnectionPool.urlopen[0;34m(self, method, url, body, headers, retries, redirect, assert_same_host, timeout, pool_timeout, release_conn, chunked, body_pos, **response_kw)[0m
[1;32m    700[0m     [39mself[39m[39m.[39m_prepare_proxy(conn)
[1;32m    702[0m [39m# Make the request on the httplib connection object.[39;00m
[0;32m--> 703[0m httplib_response [39m=[39m [39mself[39;49m[39m.[39;49m_make_request(
[1;32m    704[0m     conn,
[1;32m    705[0m     method,
[1;32m    706[0m     url,
[1;32m    707[0m     timeout[39m=[39;49mtimeout_obj,
[1;32m    708[0m     body[39m=[39;49mbody,
[1;32m    709[0m     headers[39m=[39;49mheaders,
[1;32m    710[0m     chunked[39m=[39;49mchunked,
[1;32m    711[0m )
[1;32m    713[0m [39m# If we're going to release the connection in ``finally:``, then[39;00m
[1;32m    714[0m [39m# the response doesn't need to know about the connection. Otherwise[39;00m
[1;32m    715[0m [39m# it will also try to release it and we'll have a double-release[39;00m
[1;32m    716[0m [39m# mess.[39;00m
[1;32m    717[0m response_conn [39m=[39m conn [39mif[39;00m [39mnot[39;00m release_conn [39melse[39;00m [39mNone[39;00m
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/urllib3/connectionpool.py:449[0m, in [0;36mHTTPConnectionPool._make_request[0;34m(self, conn, method, url, timeout, chunked, **httplib_request_kw)[0m
[1;32m    444[0m             httplib_response [39m=[39m conn[39m.[39mgetresponse()
[1;32m    445[0m         [39mexcept[39;00m [39mBaseException[39;00m [39mas[39;00m e:
[1;32m    446[0m             [39m# Remove the TypeError from the exception chain in[39;00m
[1;32m    447[0m             [39m# Python 3 (including for exceptions like SystemExit).[39;00m
[1;32m    448[0m             [39m# Otherwise it looks like a bug in the code.[39;00m
[0;32m--> 449[0m             six[39m.[39;49mraise_from(e, [39mNone[39;49;00m)
[1;32m    450[0m [39mexcept[39;00m (SocketTimeout, BaseSSLError, SocketError) [39mas[39;00m e:
[1;32m    451[0m     [39mself[39m[39m.[39m_raise_timeout(err[39m=[39me, url[39m=[39murl, timeout_value[39m=[39mread_timeout)
File [0;32m<string>:3[0m, in [0;36mraise_from[0;34m(value, from_value)[0m
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/urllib3/connectionpool.py:444[0m, in [0;36mHTTPConnectionPool._make_request[0;34m(self, conn, method, url, timeout, chunked, **httplib_request_kw)[0m
[1;32m    441[0m [39mexcept[39;00m [39mTypeError[39;00m:
[1;32m    442[0m     [39m# Python 3[39;00m
[1;32m    443[0m     [39mtry[39;00m:
[0;32m--> 444[0m         httplib_response [39m=[39m conn[39m.[39;49mgetresponse()
[1;32m    445[0m     [39mexcept[39;00m [39mBaseException[39;00m [39mas[39;00m e:
[1;32m    446[0m         [39m# Remove the TypeError from the exception chain in[39;00m
[1;32m    447[0m         [39m# Python 3 (including for exceptions like SystemExit).[39;00m
[1;32m    448[0m         [39m# Otherwise it looks like a bug in the code.[39;00m
[1;32m    449[0m         six[39m.[39mraise_from(e, [39mNone[39;00m)
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/http/client.py:1377[0m, in [0;36mHTTPConnection.getresponse[0;34m(self)[0m
[1;32m   1375[0m [39mtry[39;00m:
[1;32m   1376[0m     [39mtry[39;00m:
[0;32m-> 1377[0m         response[39m.[39;49mbegin()
[1;32m   1378[0m     [39mexcept[39;00m [39mConnectionError[39;00m:
[1;32m   1379[0m         [39mself[39m[39m.[39mclose()
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/http/client.py:320[0m, in [0;36mHTTPResponse.begin[0;34m(self)[0m
[1;32m    318[0m [39m# read until we get a non-100 response[39;00m
[1;32m    319[0m [39mwhile[39;00m [39mTrue[39;00m:
[0;32m--> 320[0m     version, status, reason [39m=[39m [39mself[39;49m[39m.[39;49m_read_status()
[1;32m    321[0m     [39mif[39;00m status [39m!=[39m CONTINUE:
[1;32m    322[0m         [39mbreak[39;00m
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/http/client.py:281[0m, in [0;36mHTTPResponse._read_status[0;34m(self)[0m
[1;32m    280[0m [39mdef[39;00m [39m_read_status[39m([39mself[39m):
[0;32m--> 281[0m     line [39m=[39m [39mstr[39m([39mself[39;49m[39m.[39;49mfp[39m.[39;49mreadline(_MAXLINE [39m+[39;49m [39m1[39;49m), [39m"[39m[39miso-8859-1[39m[39m"[39m)
[1;32m    282[0m     [39mif[39;00m [39mlen[39m(line) [39m>[39m _MAXLINE:
[1;32m    283[0m         [39mraise[39;00m LineTooLong([39m"[39m[39mstatus line[39m[39m"[39m)
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/socket.py:704[0m, in [0;36mSocketIO.readinto[0;34m(self, b)[0m
[1;32m    702[0m [39mwhile[39;00m [39mTrue[39;00m:
[1;32m    703[0m     [39mtry[39;00m:
[0;32m--> 704[0m         [39mreturn[39;00m [39mself[39;49m[39m.[39;49m_sock[39m.[39;49mrecv_into(b)
[1;32m    705[0m     [39mexcept[39;00m timeout:
[1;32m    706[0m         [39mself[39m[39m.[39m_timeout_occurred [39m=[39m [39mTrue[39;00m
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/ssl.py:1242[0m, in [0;36mSSLSocket.recv_into[0;34m(self, buffer, nbytes, flags)[0m
[1;32m   1238[0m     [39mif[39;00m flags [39m!=[39m [39m0[39m:
[1;32m   1239[0m         [39mraise[39;00m [39mValueError[39;00m(
[1;32m   1240[0m           [39m"[39m[39mnon-zero flags not allowed in calls to recv_into() on [39m[39m%s[39;00m[39m"[39m [39m%[39m
[1;32m   1241[0m           [39mself[39m[39m.[39m[39m__class__[39m)
[0;32m-> 1242[0m     [39mreturn[39;00m [39mself[39;49m[39m.[39;49mread(nbytes, buffer)
[1;32m   1243[0m [39melse[39;00m:
[1;32m   1244[0m     [39mreturn[39;00m [39msuper[39m()[39m.[39mrecv_into(buffer, nbytes, flags)
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/ssl.py:1100[0m, in [0;36mSSLSocket.read[0;34m(self, len, buffer)[0m
[1;32m   1098[0m [39mtry[39;00m:
[1;32m   1099[0m     [39mif[39;00m buffer [39mis[39;00m [39mnot[39;00m [39mNone[39;00m:
[0;32m-> 1100[0m         [39mreturn[39;00m [39mself[39;49m[39m.[39;49m_sslobj[39m.[39;49mread([39mlen[39;49m, buffer)
[1;32m   1101[0m     [39melse[39;00m:
[1;32m   1102[0m         [39mreturn[39;00m [39mself[39m[39m.[39m_sslobj[39m.[39mread([39mlen[39m)
[0;31mKeyboardInterrupt[0m:
```

## Query the Database
With our data safely inserted in Milvus, we can now perform a query. The query takes in a string or a list of strings and searches them. The output includes your query and each result's score, title, and book description. 

```python
import textwrap

def query(queries, top_k = 5):
    if type(queries) != list:
        queries = [queries]
    res = collection.search(embed(queries), anns_field='embedding', param=QUERY_PARAM, limit = top_k, output_fields=['title', 'description'])
    for i, hit in enumerate(res):
        print('Description:', queries[i])
        print('Results:')
        for ii, hits in enumerate(hit):
            print('\t' + 'Rank:', ii + 1, 'Score:', hits.score, 'Title:', hits.entity.get('title'))
            print(textwrap.fill(hits.entity.get('description'), 88))
            print()
```

```python
query('Book about a k-9 from europe')
```

```text
RPC error: [search], <MilvusException: (code=1, message=code: UnexpectedError, reason: code: CollectionNotExists, reason: can't find collection: book_search)>, <Time:{'RPC start': '2023-03-17 14:22:18.368461', 'RPC error': '2023-03-17 14:22:18.382086'}>
```

```text
MilvusException
<MilvusException: (code=1, message=code: UnexpectedError, reason: code: CollectionNotExists, reason: can't find collection: book_search)>
[0;31m---------------------------------------------------------------------------[0m[0;31mMilvusException[0m                           Traceback (most recent call last)Cell [0;32mIn[32], line 1[0m
[0;32m----> 1[0m query([39m'[39;49m[39mBook about a k-9 from europe[39;49m[39m'[39;49m)
Cell [0;32mIn[31], line 6[0m, in [0;36mquery[0;34m(queries, top_k)[0m
[1;32m      4[0m [39mif[39;00m [39mtype[39m(queries) [39m!=[39m [39mlist[39m:
[1;32m      5[0m     queries [39m=[39m [queries]
[0;32m----> 6[0m res [39m=[39m collection[39m.[39;49msearch(embed(queries), anns_field[39m=[39;49m[39m'[39;49m[39membedding[39;49m[39m'[39;49m, param[39m=[39;49mQUERY_PARAM, limit [39m=[39;49m top_k, output_fields[39m=[39;49m[[39m'[39;49m[39mtitle[39;49m[39m'[39;49m, [39m'[39;49m[39mdescription[39;49m[39m'[39;49m])
[1;32m      7[0m [39mfor[39;00m i, hit [39min[39;00m [39menumerate[39m(res):
[1;32m      8[0m     [39mprint[39m([39m'[39m[39mDescription:[39m[39m'[39m, queries[i])
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/orm/collection.py:614[0m, in [0;36mCollection.search[0;34m(self, data, anns_field, param, limit, expr, partition_names, output_fields, timeout, round_decimal, **kwargs)[0m
[1;32m    611[0m     [39mraise[39;00m DataTypeNotMatchException(message[39m=[39mExceptionsMessage[39m.[39mExprType [39m%[39m [39mtype[39m(expr))
[1;32m    613[0m conn [39m=[39m [39mself[39m[39m.[39m_get_connection()
[0;32m--> 614[0m res [39m=[39m conn[39m.[39;49msearch([39mself[39;49m[39m.[39;49m_name, data, anns_field, param, limit, expr,
[1;32m    615[0m                   partition_names, output_fields, round_decimal, timeout[39m=[39;49mtimeout,
[1;32m    616[0m                   schema[39m=[39;49m[39mself[39;49m[39m.[39;49m_schema_dict, [39m*[39;49m[39m*[39;49mkwargs)
[1;32m    617[0m [39mif[39;00m kwargs[39m.[39mget([39m"[39m[39m_async[39m[39m"[39m, [39mFalse[39;00m):
[1;32m    618[0m     [39mreturn[39;00m SearchFuture(res)
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/decorators.py:109[0m, in [0;36merror_handler.<locals>.wrapper.<locals>.handler[0;34m(*args, **kwargs)[0m
[1;32m    107[0m     record_dict[[39m"[39m[39mRPC error[39m[39m"[39m] [39m=[39m [39mstr[39m(datetime[39m.[39mdatetime[39m.[39mnow())
[1;32m    108[0m     LOGGER[39m.[39merror([39mf[39m[39m"[39m[39mRPC error: [[39m[39m{[39;00minner_name[39m}[39;00m[39m], [39m[39m{[39;00me[39m}[39;00m[39m, <Time:[39m[39m{[39;00mrecord_dict[39m}[39;00m[39m>[39m[39m"[39m)
[0;32m--> 109[0m     [39mraise[39;00m e
[1;32m    110[0m [39mexcept[39;00m grpc[39m.[39mFutureTimeoutError [39mas[39;00m e:
[1;32m    111[0m     record_dict[[39m"[39m[39mgRPC timeout[39m[39m"[39m] [39m=[39m [39mstr[39m(datetime[39m.[39mdatetime[39m.[39mnow())
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/decorators.py:105[0m, in [0;36merror_handler.<locals>.wrapper.<locals>.handler[0;34m(*args, **kwargs)[0m
[1;32m    103[0m [39mtry[39;00m:
[1;32m    104[0m     record_dict[[39m"[39m[39mRPC start[39m[39m"[39m] [39m=[39m [39mstr[39m(datetime[39m.[39mdatetime[39m.[39mnow())
[0;32m--> 105[0m     [39mreturn[39;00m func([39m*[39;49margs, [39m*[39;49m[39m*[39;49mkwargs)
[1;32m    106[0m [39mexcept[39;00m MilvusException [39mas[39;00m e:
[1;32m    107[0m     record_dict[[39m"[39m[39mRPC error[39m[39m"[39m] [39m=[39m [39mstr[39m(datetime[39m.[39mdatetime[39m.[39mnow())
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/decorators.py:136[0m, in [0;36mtracing_request.<locals>.wrapper.<locals>.handler[0;34m(self, *args, **kwargs)[0m
[1;32m    134[0m [39mif[39;00m req_id:
[1;32m    135[0m     [39mself[39m[39m.[39mset_onetime_request_id(req_id)
[0;32m--> 136[0m ret [39m=[39m func([39mself[39;49m, [39m*[39;49margs, [39m*[39;49m[39m*[39;49mkwargs)
[1;32m    137[0m [39mreturn[39;00m ret
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/decorators.py:85[0m, in [0;36mretry_on_rpc_failure.<locals>.wrapper.<locals>.handler[0;34m(self, *args, **kwargs)[0m
[1;32m     83[0m         back_off [39m=[39m [39mmin[39m(back_off [39m*[39m back_off_multiplier, max_back_off)
[1;32m     84[0m     [39melse[39;00m:
[0;32m---> 85[0m         [39mraise[39;00m e
[1;32m     86[0m [39mexcept[39;00m [39mException[39;00m [39mas[39;00m e:
[1;32m     87[0m     [39mraise[39;00m e
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/decorators.py:50[0m, in [0;36mretry_on_rpc_failure.<locals>.wrapper.<locals>.handler[0;34m(self, *args, **kwargs)[0m
[1;32m     48[0m [39mwhile[39;00m [39mTrue[39;00m:
[1;32m     49[0m     [39mtry[39;00m:
[0;32m---> 50[0m         [39mreturn[39;00m func([39mself[39;49m, [39m*[39;49margs, [39m*[39;49m[39m*[39;49mkwargs)
[1;32m     51[0m     [39mexcept[39;00m grpc[39m.[39mRpcError [39mas[39;00m e:
[1;32m     52[0m         [39m# DEADLINE_EXCEEDED means that the task wat not completed[39;00m
[1;32m     53[0m         [39m# UNAVAILABLE means that the service is not reachable currently[39;00m
[1;32m     54[0m         [39m# Reference: https://grpc.github.io/grpc/python/grpc.html#grpc-status-code[39;00m
[1;32m     55[0m         [39mif[39;00m e[39m.[39mcode() [39m!=[39m grpc[39m.[39mStatusCode[39m.[39mDEADLINE_EXCEEDED [39mand[39;00m e[39m.[39mcode() [39m!=[39m grpc[39m.[39mStatusCode[39m.[39mUNAVAILABLE:
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/client/grpc_handler.py:472[0m, in [0;36mGrpcHandler.search[0;34m(self, collection_name, data, anns_field, param, limit, expression, partition_names, output_fields, round_decimal, timeout, schema, **kwargs)[0m
[1;32m    467[0m requests [39m=[39m Prepare[39m.[39msearch_requests_with_expr(collection_name, data, anns_field, param, limit, schema,
[1;32m    468[0m                                              expression, partition_names, output_fields, round_decimal,
[1;32m    469[0m                                              [39m*[39m[39m*[39mkwargs)
[1;32m    471[0m auto_id [39m=[39m schema[[39m"[39m[39mauto_id[39m[39m"[39m]
[0;32m--> 472[0m [39mreturn[39;00m [39mself[39;49m[39m.[39;49m_execute_search_requests(requests, timeout, round_decimal[39m=[39;49mround_decimal, auto_id[39m=[39;49mauto_id, [39m*[39;49m[39m*[39;49mkwargs)
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/client/grpc_handler.py:441[0m, in [0;36mGrpcHandler._execute_search_requests[0;34m(self, requests, timeout, **kwargs)[0m
[1;32m    439[0m [39mif[39;00m kwargs[39m.[39mget([39m"[39m[39m_async[39m[39m"[39m, [39mFalse[39;00m):
[1;32m    440[0m     [39mreturn[39;00m SearchFuture([39mNone[39;00m, [39mNone[39;00m, [39mTrue[39;00m, pre_err)
[0;32m--> 441[0m [39mraise[39;00m pre_err
File [0;32m~/miniconda3/envs/haystack/lib/python3.9/site-packages/pymilvus/client/grpc_handler.py:432[0m, in [0;36mGrpcHandler._execute_search_requests[0;34m(self, requests, timeout, **kwargs)[0m
[1;32m    429[0m     response [39m=[39m [39mself[39m[39m.[39m_stub[39m.[39mSearch(request, timeout[39m=[39mtimeout)
[1;32m    431[0m     [39mif[39;00m response[39m.[39mstatus[39m.[39merror_code [39m!=[39m [39m0[39m:
[0;32m--> 432[0m         [39mraise[39;00m MilvusException(response[39m.[39mstatus[39m.[39merror_code, response[39m.[39mstatus[39m.[39mreason)
[1;32m    434[0m     raws[39m.[39mappend(response)
[1;32m    435[0m round_decimal [39m=[39m kwargs[39m.[39mget([39m"[39m[39mround_decimal[39m[39m"[39m, [39m-[39m[39m1[39m)
[0;31mMilvusException[0m: <MilvusException: (code=1, message=code: UnexpectedError, reason: code: CollectionNotExists, reason: can't find collection: book_search)>
```