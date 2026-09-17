# Using Redis for Embeddings Search

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook takes you through a simple flow to download some data, embed it, and then index and search it using a selection of vector databases. This is a common requirement for customers who want to store and search our embeddings with their own data in a secure environment to support production use cases such as chatbots, topic modelling and more.

### What is a Vector Database

A vector database is a database made to store, manage and search embedding vectors. The use of embeddings to encode unstructured data (text, audio, video and more) as vectors for consumption by machine-learning models has exploded in recent years, due to the increasing effectiveness of AI in solving use cases involving natural language, image recognition and other unstructured forms of data. Vector databases have emerged as an effective solution for enterprises to deliver and scale these use cases.

### Why use a Vector Database

Vector databases enable enterprises to take many of the embeddings use cases we've shared in this repo (question and answering, chatbot and recommendation services, for example), and make use of them in a secure, scalable environment. Many of our customers make embeddings solve their problems at small scale but performance and security hold them back from going into production - we see vector databases as a key component in solving that, and in this guide we'll walk through the basics of embedding text data, storing it in a vector database and using it for semantic search.


### Demo Flow
The demo flow is:
- **Setup**: Import packages and set any required variables
- **Load data**: Load a dataset and embed it using OpenAI embeddings
- **Redis**
    - *Setup*: Set up the Redis-Py client. For more details go [here](https://github.com/redis/redis-py)
    - *Index Data*: Create the search index for vector search and hybrid search (vector + full-text search) on all available fields.
    - *Search Data*: Run a few example queries with various goals in mind.

Once you've run through this notebook you should have a basic understanding of how to setup and use vector databases, and can move on to more complex use cases making use of our embeddings.

## Setup

Import the required libraries and set the embedding model that we'd like to use.

```python
# We'll need to install the Redis client
!pip install redis

#Install wget to pull zip file
!pip install wget
```

```python
import openai

from typing import List, Iterator
import pandas as pd
import numpy as np
import os
import wget
from ast import literal_eval

# Redis client library for Python
import redis

# I've set this to our new embeddings model, this can be changed to the embedding model of your choice
EMBEDDING_MODEL = "text-embedding-3-small"

# Ignore unclosed SSL socket warnings - optional in case you get these errors
import warnings

warnings.filterwarnings(action="ignore", message="unclosed", category=ResourceWarning)
warnings.filterwarnings("ignore", category=DeprecationWarning)
```

## Load data

In this section we'll load embedded data that we've prepared previous to this session.

```python
embeddings_url = 'https://cdn.openai.com/API/examples/data/vector_database_wikipedia_articles_embedded.zip'

# The file is ~700 MB so this will take some time
wget.download(embeddings_url)
```

```python
import zipfile
with zipfile.ZipFile("vector_database_wikipedia_articles_embedded.zip","r") as zip_ref:
    zip_ref.extractall("../data")
```

```python
article_df = pd.read_csv('../data/vector_database_wikipedia_articles_embedded.csv')
```

```python
article_df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>url</th>
      <th>title</th>
      <th>text</th>
      <th>title_vector</th>
      <th>content_vector</th>
      <th>vector_id</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>https://simple.wikipedia.org/wiki/April</td>
      <td>April</td>
      <td>April is the fourth month of the year in the J...</td>
      <td>[0.001009464613161981, -0.020700545981526375, ...</td>
      <td>[-0.011253940872848034, -0.013491976074874401,...</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>https://simple.wikipedia.org/wiki/August</td>
      <td>August</td>
      <td>August (Aug.) is the eighth month of the year ...</td>
      <td>[0.0009286514250561595, 0.000820168002974242, ...</td>
      <td>[0.0003609954728744924, 0.007262262050062418, ...</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>https://simple.wikipedia.org/wiki/Art</td>
      <td>Art</td>
      <td>Art is a creative activity that expresses imag...</td>
      <td>[0.003393713850528002, 0.0061537534929811954, ...</td>
      <td>[-0.004959689453244209, 0.015772193670272827, ...</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>8</td>
      <td>https://simple.wikipedia.org/wiki/A</td>
      <td>A</td>
      <td>A or a is the first letter of the English alph...</td>
      <td>[0.0153952119871974, -0.013759135268628597, 0....</td>
      <td>[0.024894846603274345, -0.022186409682035446, ...</td>
      <td>3</td>
    </tr>
    <tr>
      <th>4</th>
      <td>9</td>
      <td>https://simple.wikipedia.org/wiki/Air</td>
      <td>Air</td>
      <td>Air refers to the Earth's atmosphere. Air is a...</td>
      <td>[0.02224554680287838, -0.02044147066771984, -0...</td>
      <td>[0.021524671465158463, 0.018522677943110466, -...</td>
      <td>4</td>
    </tr>
  </tbody>
</table>
</div>

```python
# Read vectors from strings back into a list
article_df['title_vector'] = article_df.title_vector.apply(literal_eval)
article_df['content_vector'] = article_df.content_vector.apply(literal_eval)

# Set vector_id to be a string
article_df['vector_id'] = article_df['vector_id'].apply(str)
```

```python
article_df.info(show_counts=True)
```

```text
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 25000 entries, 0 to 24999
Data columns (total 7 columns):
 #   Column          Non-Null Count  Dtype 
---  ------          --------------  ----- 
 0   id              25000 non-null  int64 
 1   url             25000 non-null  object
 2   title           25000 non-null  object
 3   text            25000 non-null  object
 4   title_vector    25000 non-null  object
 5   content_vector  25000 non-null  object
 6   vector_id       25000 non-null  object
dtypes: int64(1), object(6)
memory usage: 1.3+ MB
```

# Redis

The next vector database covered in this tutorial is **[Redis](https://redis.io)**. You most likely already know Redis. What you might not be aware of is the [RediSearch module](https://github.com/RediSearch/RediSearch). Enterprises have been using Redis with the RediSearch module for years now across all major cloud providers, Redis Cloud, and on premise. Recently, the Redis team added vector storage and search capability to this module in addition to the features RediSearch already had.

Given the large ecosystem around Redis, there are most likely client libraries in the language you need. You can use any standard Redis client library to run RediSearch commands, but it's easiest to use a library that wraps the RediSearch API. Below are a few examples, but you can find more client libraries [here](https://redis.io/resources/clients/).

| Project | Language | License | Author | Stars |
|----------|---------|--------|---------|-------|
| [jedis][jedis-url] | Java | MIT | [Redis][redis-url] | ![Stars][jedis-stars] |
| [redis-py][redis-py-url] | Python | MIT | [Redis][redis-url] | ![Stars][redis-py-stars] |
| [node-redis][node-redis-url] | Node.js | MIT | [Redis][redis-url] | ![Stars][node-redis-stars] |
| [nredisstack][nredisstack-url] | .NET | MIT | [Redis][redis-url] | ![Stars][nredisstack-stars] |
| [redisearch-go][redisearch-go-url] | Go | BSD | [Redis][redisearch-go-author] | [![redisearch-go-stars]][redisearch-go-url] |
| [redisearch-api-rs][redisearch-api-rs-url] | Rust | BSD | [Redis][redisearch-api-rs-author] | [![redisearch-api-rs-stars]][redisearch-api-rs-url] |

[redis-url]: https://redis.com
[redis-py-url]: https://github.com/redis/redis-py
[redis-py-stars]: https://img.shields.io/github/stars/redis/redis-py.svg?style=social&amp;label=Star&amp;maxAge=2592000
[redis-py-package]: https://pypi.python.org/pypi/redis
[jedis-url]: https://github.com/redis/jedis
[jedis-stars]: https://img.shields.io/github/stars/redis/jedis.svg?style=social&amp;label=Star&amp;maxAge=2592000
[Jedis-package]: https://search.maven.org/artifact/redis.clients/jedis
[nredisstack-url]: https://github.com/redis/nredisstack
[nredisstack-stars]: https://img.shields.io/github/stars/redis/nredisstack.svg?style=social&amp;label=Star&amp;maxAge=2592000
[nredisstack-package]: https://www.nuget.org/packages/nredisstack/
[node-redis-url]: https://github.com/redis/node-redis
[node-redis-stars]: https://img.shields.io/github/stars/redis/node-redis.svg?style=social&amp;label=Star&amp;maxAge=2592000
[node-redis-package]: https://www.npmjs.com/package/redis
[redis-om-python-url]: https://github.com/redis/redis-om-python
[redis-om-python-author]: https://redis.com
[redis-om-python-stars]: https://img.shields.io/github/stars/redis/redis-om-python.svg?style=social&amp;label=Star&amp;maxAge=2592000
[redisearch-go-url]: https://github.com/RediSearch/redisearch-go
[redisearch-go-author]: https://redis.com
[redisearch-go-stars]: https://img.shields.io/github/stars/RediSearch/redisearch-go.svg?style=social&amp;label=Star&amp;maxAge=2592000
[redisearch-api-rs-url]: https://github.com/RediSearch/redisearch-api-rs
[redisearch-api-rs-author]: https://redis.com
[redisearch-api-rs-stars]: https://img.shields.io/github/stars/RediSearch/redisearch-api-rs.svg?style=social&amp;label=Star&amp;maxAge=2592000
In the below cells, we will walk you through using Redis as a vector database. Since many of you are likely already used to the Redis API, this should be familiar to most.

## Setup

There are many ways to deploy Redis with RediSearch. The easiest way to get started is to use Docker, but there are many potential options for deployment. For other deployment options, see the [Redis directory](https://developers.openai.com/cookbook/examples/vector_databases/redis/) in this repo.

For this tutorial, we will use Redis Stack on Docker.

Start a version of Redis with RediSearch (Redis Stack) by running the following docker command

```bash
$ cd redis
$ docker compose up -d
```
This also includes the [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) GUI for managing your Redis database which you can view at [http://localhost:8001](http://localhost:8001) once you start the docker container.

You're all set up and ready to go! Next, we import and create our client for communicating with the Redis database we just created.

```python
import redis
from redis.commands.search.indexDefinition import (
    IndexDefinition,
    IndexType
)
from redis.commands.search.query import Query
from redis.commands.search.field import (
    TextField,
    VectorField
)

REDIS_HOST =  "localhost"
REDIS_PORT = 6379
REDIS_PASSWORD = "" # default for passwordless Redis

# Connect to Redis
redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=REDIS_PASSWORD
)
redis_client.ping()
```

```text
True
```

## Creating a Search Index

The below cells will show how to specify and create a search index in Redis. We will

1. Set some constants for defining our index like the distance metric and the index name
2. Define the index schema with RediSearch fields
3. Create the index


```python
# Constants
VECTOR_DIM = len(article_df['title_vector'][0]) # length of the vectors
VECTOR_NUMBER = len(article_df)                 # initial number of vectors
INDEX_NAME = "embeddings-index"                 # name of the search index
PREFIX = "doc"                                  # prefix for the document keys
DISTANCE_METRIC = "COSINE"                      # distance metric for the vectors (ex. COSINE, IP, L2)
```

```python
# Define RediSearch fields for each of the columns in the dataset
title = TextField(name="title")
url = TextField(name="url")
text = TextField(name="text")
title_embedding = VectorField("title_vector",
    "FLAT", {
        "TYPE": "FLOAT32",
        "DIM": VECTOR_DIM,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": VECTOR_NUMBER,
    }
)
text_embedding = VectorField("content_vector",
    "FLAT", {
        "TYPE": "FLOAT32",
        "DIM": VECTOR_DIM,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": VECTOR_NUMBER,
    }
)
fields = [title, url, text, title_embedding, text_embedding]
```

```python
# Check if index exists
try:
    redis_client.ft(INDEX_NAME).info()
    print("Index already exists")
except:
    # Create RediSearch Index
    redis_client.ft(INDEX_NAME).create_index(
        fields = fields,
        definition = IndexDefinition(prefix=[PREFIX], index_type=IndexType.HASH)
    )
```

## Load Documents into the Index

Now that we have a search index, we can load documents into it. We will use the same documents we used in the previous examples. In Redis, either the Hash or JSON (if using RedisJSON in addition to RediSearch) data types can be used to store documents. We will use the HASH data type in this example. The below cells will show how to load documents into the index.

```python
def index_documents(client: redis.Redis, prefix: str, documents: pd.DataFrame):
    records = documents.to_dict("records")
    for doc in records:
        key = f"{prefix}:{str(doc['id'])}"

        # create byte vectors for title and content
        title_embedding = np.array(doc["title_vector"], dtype=np.float32).tobytes()
        content_embedding = np.array(doc["content_vector"], dtype=np.float32).tobytes()

        # replace list of floats with byte vectors
        doc["title_vector"] = title_embedding
        doc["content_vector"] = content_embedding

        client.hset(key, mapping = doc)
```

```python
index_documents(redis_client, PREFIX, article_df)
print(f"Loaded {redis_client.info()['db0']['keys']} documents in Redis search index with name: {INDEX_NAME}")
```

```text
Loaded 25000 documents in Redis search index with name: embeddings-index
```

## Running Search Queries

Now that we have a search index and documents loaded into it, we can run search queries. Below we will provide a function that will run a search query and return the results. Using this function we run a few queries that will show how you can utilize Redis as a vector database. Each example will demonstrate specific features to keep in mind when developing your search application with Redis.

1. **Return Fields**: You can specify which fields you want to return in the search results. This is useful if you only want to return a subset of the fields in your documents and doesn't require a separate call to retrieve documents. In the below example, we will only return the `title` field in the search results.
2. **Hybrid Search**: You can combine vector search with any of the other RediSearch fields for hybrid search such as full text search, tag, geo, and numeric. In the below example, we will combine vector search with full text search.


```python
def search_redis(
    redis_client: redis.Redis,
    user_query: str,
    index_name: str = "embeddings-index",
    vector_field: str = "title_vector",
    return_fields: list = ["title", "url", "text", "vector_score"],
    hybrid_fields = "*",
    k: int = 20,
) -> List[dict]:

    # Creates embedding vector from user query
    embedded_query = openai.Embedding.create(input=user_query,
                                            model=EMBEDDING_MODEL,
                                            )["data"][0]['embedding']

    # Prepare the Query
    base_query = f'{hybrid_fields}=>[KNN {k} @{vector_field} $vector AS vector_score]'
    query = (
        Query(base_query)
         .return_fields(*return_fields)
         .sort_by("vector_score")
         .paging(0, k)
         .dialect(2)
    )
    params_dict = {"vector": np.array(embedded_query).astype(dtype=np.float32).tobytes()}

    # perform vector search
    results = redis_client.ft(index_name).search(query, params_dict)
    for i, article in enumerate(results.docs):
        score = 1 - float(article.vector_score)
        print(f"{i}. {article.title} (Score: {round(score ,3) })")
    return results.docs
```

```python
# For using OpenAI to generate query embedding
openai.api_key = os.getenv("OPENAI_API_KEY", "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")
results = search_redis(redis_client, 'modern art in Europe', k=10)
```

```text
0. Museum of Modern Art (Score: 0.875)
1. Western Europe (Score: 0.867)
2. Renaissance art (Score: 0.864)
3. Pop art (Score: 0.86)
4. Northern Europe (Score: 0.855)
5. Hellenistic art (Score: 0.853)
6. Modernist literature (Score: 0.847)
7. Art film (Score: 0.843)
8. Central Europe (Score: 0.843)
9. European (Score: 0.841)
```

```python
results = search_redis(redis_client, 'Famous battles in Scottish history', vector_field='content_vector', k=10)
```

```text
0. Battle of Bannockburn (Score: 0.869)
1. Wars of Scottish Independence (Score: 0.861)
2. 1651 (Score: 0.853)
3. First War of Scottish Independence (Score: 0.85)
4. Robert I of Scotland (Score: 0.846)
5. 841 (Score: 0.844)
6. 1716 (Score: 0.844)
7. 1314 (Score: 0.837)
8. 1263 (Score: 0.836)
9. William Wallace (Score: 0.835)
```

## Hybrid Queries with Redis

The previous examples showed how run vector search queries with RediSearch. In this section, we will show how to combine vector search with other RediSearch fields for hybrid search. In the below example, we will combine vector search with full text search.

```python
def create_hybrid_field(field_name: str, value: str) -> str:
    return f'@{field_name}:"{value}"'
```

```python
# search the content vector for articles about famous battles in Scottish history and only include results with Scottish in the title
results = search_redis(redis_client,
                       "Famous battles in Scottish history",
                       vector_field="title_vector",
                       k=5,
                       hybrid_fields=create_hybrid_field("title", "Scottish")
                       )
```

```text
0. First War of Scottish Independence (Score: 0.892)
1. Wars of Scottish Independence (Score: 0.889)
2. Second War of Scottish Independence (Score: 0.879)
3. List of Scottish monarchs (Score: 0.873)
4. Scottish Borders (Score: 0.863)
```

```python
# run a hybrid query for articles about Art in the title vector and only include results with the phrase "Leonardo da Vinci" in the text
results = search_redis(redis_client,
                       "Art",
                       vector_field="title_vector",
                       k=5,
                       hybrid_fields=create_hybrid_field("text", "Leonardo da Vinci")
                       )

# find specific mention of Leonardo da Vinci in the text that our full-text-search query returned
mention = [sentence for sentence in results[0].text.split("\n") if "Leonardo da Vinci" in sentence][0]
mention
```

```text
0. Art (Score: 1.0)
1. Paint (Score: 0.896)
2. Renaissance art (Score: 0.88)
3. Painting (Score: 0.874)
4. Renaissance (Score: 0.846)
```

```text
'In Europe, after the Middle Ages, there was a "Renaissance" which means "rebirth". People rediscovered science and artists were allowed to paint subjects other than religious subjects. People like Michelangelo and Leonardo da Vinci still painted religious pictures, but they also now could paint mythological pictures too. These artists also invented perspective where things in the distance look smaller in the picture. This was new because in the Middle Ages people would paint all the figures close up and just overlapping each other. These artists used nudity regularly in their art.'
```

For more example with Redis as a vector database, see the README and examples within the ``vector_databases/redis`` directory of this repository