# Running Hybrid VSS Queries with Redis and OpenAI

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook provides an introduction to using Redis as a vector database with OpenAI embeddings and running hybrid queries that combine VSS and lexical search using Redis Query and Search capability. Redis is a scalable, real-time database that can be used as a vector database when using the [RediSearch Module](https://oss.redislabs.com/redisearch/). The Redis Query and Search capability allows you to index and search for vectors in Redis. This notebook will show you how to use the Redis Query and Search to index and search for vectors created by using the OpenAI API and stored in Redis.

Hybrid queries combine vector similarity with traditional Redis Query and Search filtering capabilities on GEO, NUMERIC, TAG or TEXT data simplifying application code. A common example of a hybrid query in an e-commerce use case is to find items visually similar to a given query image limited to items available in a GEO location and within a price range.

## Prerequisites

Before we start this project, we need to set up the following:

* start a Redis database with RediSearch (redis-stack)
* install libraries
    * [Redis-py](https://github.com/redis/redis-py)
* get your [OpenAI API key](https://beta.openai.com/account/api-keys)

===========================================================

### Start Redis

To keep this example simple, we will use the Redis Stack docker container which we can start as follows

```bash
$ docker-compose up -d
```

This also includes the [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) GUI for managing your Redis database which you can view at [http://localhost:8001](http://localhost:8001) once you start the docker container.

You're all set up and ready to go! Next, we import and create our client for communicating with the Redis database we just created.

## Install Requirements

Redis-Py is the python client for communicating with Redis. We will use this to communicate with our Redis-stack database. 

```python
! pip install redis pandas openai
```

```text
Defaulting to user installation because normal site-packages is not writeable
Requirement already satisfied: redis in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (4.5.4)
Requirement already satisfied: pandas in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (2.0.1)
Requirement already satisfied: openai in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (0.27.6)
Requirement already satisfied: async-timeout>=4.0.2 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from redis) (4.0.2)
Requirement already satisfied: python-dateutil>=2.8.2 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from pandas) (2.8.2)
Requirement already satisfied: pytz>=2020.1 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from pandas) (2023.3)
Requirement already satisfied: tzdata>=2022.1 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from pandas) (2023.3)
Requirement already satisfied: numpy>=1.20.3 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from pandas) (1.23.4)
Requirement already satisfied: requests>=2.20 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from openai) (2.28.1)
Requirement already satisfied: tqdm in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from openai) (4.64.1)
Requirement already satisfied: aiohttp in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from openai) (3.8.4)
Requirement already satisfied: six>=1.5 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from python-dateutil>=2.8.2->pandas) (1.16.0)
Requirement already satisfied: charset-normalizer<3,>=2 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from requests>=2.20->openai) (2.1.1)
Requirement already satisfied: idna<4,>=2.5 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from requests>=2.20->openai) (3.4)
Requirement already satisfied: urllib3<1.27,>=1.21.1 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from requests>=2.20->openai) (1.26.12)
Requirement already satisfied: certifi>=2017.4.17 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from requests>=2.20->openai) (2022.9.24)
Requirement already satisfied: attrs>=17.3.0 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from aiohttp->openai) (23.1.0)
Requirement already satisfied: multidict<7.0,>=4.5 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from aiohttp->openai) (6.0.4)
Requirement already satisfied: yarl<2.0,>=1.0 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from aiohttp->openai) (1.9.2)
Requirement already satisfied: frozenlist>=1.1.1 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from aiohttp->openai) (1.3.3)
Requirement already satisfied: aiosignal>=1.1.2 in /Users/michael.yuan/Library/Python/3.9/lib/python/site-packages (from aiohttp->openai) (1.3.1)
```

===========================================================
## Prepare your OpenAI API key

The `OpenAI API key` is used for vectorization of query data.

If you don't have an OpenAI API key, you can get one from [https://beta.openai.com/account/api-keys](https://beta.openai.com/account/api-keys).

Once you get your key, please add it to your environment variables as `OPENAI_API_KEY` by using following command:

```python
# Test that your OpenAI API key is correctly set as an environment variable
# Note. if you run this notebook locally, you will need to reload your terminal and the notebook for the env variables to be live.
import os
import openai

os.environ["OPENAI_API_KEY"] = '<YOUR_OPENAI_API_KEY>'

if os.getenv("OPENAI_API_KEY") is not None:
    openai.api_key = os.getenv("OPENAI_API_KEY")
    print ("OPENAI_API_KEY is ready")
else:
    print ("OPENAI_API_KEY environment variable not found")
```

```text
OPENAI_API_KEY is ready
```

## Load data

In this section we'll load and clean an ecommerce dataset. We'll generate embeddings using OpenAI and use this data to create an index in Redis and then search for similar vectors.

```python
import pandas as pd
import numpy as np
from typing import List

from utils.embeddings_utils import (
    get_embeddings,
    distances_from_embeddings,
    tsne_components_from_embeddings,
    chart_from_components,
    indices_of_nearest_neighbors_from_distances,
)

EMBEDDING_MODEL = "text-embedding-3-small"

# load in data and clean data types and drop null rows
df = pd.read_csv("../../data/styles_2k.csv", on_bad_lines='skip')
df.dropna(inplace=True)
df["year"] = df["year"].astype(int)
df.info()

# print dataframe
n_examples = 5
df.head(n_examples)
```

```text
<class 'pandas.core.frame.DataFrame'>
Index: 1978 entries, 0 to 1998
Data columns (total 10 columns):
 #   Column              Non-Null Count  Dtype 
---  ------              --------------  ----- 
 0   id                  1978 non-null   int64 
 1   gender              1978 non-null   object
 2   masterCategory      1978 non-null   object
 3   subCategory         1978 non-null   object
 4   articleType         1978 non-null   object
 5   baseColour          1978 non-null   object
 6   season              1978 non-null   object
 7   year                1978 non-null   int64 
 8   usage               1978 non-null   object
 9   productDisplayName  1978 non-null   object
dtypes: int64(2), object(8)
memory usage: 170.0+ KB
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>gender</th>
      <th>masterCategory</th>
      <th>subCategory</th>
      <th>articleType</th>
      <th>baseColour</th>
      <th>season</th>
      <th>year</th>
      <th>usage</th>
      <th>productDisplayName</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>15970</td>
      <td>Men</td>
      <td>Apparel</td>
      <td>Topwear</td>
      <td>Shirts</td>
      <td>Navy Blue</td>
      <td>Fall</td>
      <td>2011</td>
      <td>Casual</td>
      <td>Turtle Check Men Navy Blue Shirt</td>
    </tr>
    <tr>
      <th>1</th>
      <td>39386</td>
      <td>Men</td>
      <td>Apparel</td>
      <td>Bottomwear</td>
      <td>Jeans</td>
      <td>Blue</td>
      <td>Summer</td>
      <td>2012</td>
      <td>Casual</td>
      <td>Peter England Men Party Blue Jeans</td>
    </tr>
    <tr>
      <th>2</th>
      <td>59263</td>
      <td>Women</td>
      <td>Accessories</td>
      <td>Watches</td>
      <td>Watches</td>
      <td>Silver</td>
      <td>Winter</td>
      <td>2016</td>
      <td>Casual</td>
      <td>Titan Women Silver Watch</td>
    </tr>
    <tr>
      <th>3</th>
      <td>21379</td>
      <td>Men</td>
      <td>Apparel</td>
      <td>Bottomwear</td>
      <td>Track Pants</td>
      <td>Black</td>
      <td>Fall</td>
      <td>2011</td>
      <td>Casual</td>
      <td>Manchester United Men Solid Black Track Pants</td>
    </tr>
    <tr>
      <th>4</th>
      <td>53759</td>
      <td>Men</td>
      <td>Apparel</td>
      <td>Topwear</td>
      <td>Tshirts</td>
      <td>Grey</td>
      <td>Summer</td>
      <td>2012</td>
      <td>Casual</td>
      <td>Puma Men Grey T-shirt</td>
    </tr>
  </tbody>
</table>
</div>

```python
df["product_text"] = df.apply(lambda row: f"name {row['productDisplayName']} category {row['masterCategory']} subcategory {row['subCategory']} color {row['baseColour']} gender {row['gender']}".lower(), axis=1)
df.rename({"id":"product_id"}, inplace=True, axis=1)

df.info()
```

```text
<class 'pandas.core.frame.DataFrame'>
Index: 1978 entries, 0 to 1998
Data columns (total 11 columns):
 #   Column              Non-Null Count  Dtype 
---  ------              --------------  ----- 
 0   product_id          1978 non-null   int64 
 1   gender              1978 non-null   object
 2   masterCategory      1978 non-null   object
 3   subCategory         1978 non-null   object
 4   articleType         1978 non-null   object
 5   baseColour          1978 non-null   object
 6   season              1978 non-null   object
 7   year                1978 non-null   int64 
 8   usage               1978 non-null   object
 9   productDisplayName  1978 non-null   object
 10  product_text        1978 non-null   object
dtypes: int64(2), object(9)
memory usage: 185.4+ KB
```

```python
# check out one of the texts we will use to create semantic embeddings
df["product_text"][0]
```

```text
'name turtle check men navy blue shirt category apparel subcategory topwear color navy blue gender men'
```

## Connect to Redis

Now that we have our Redis database running, we can connect to it using the Redis-py client. We will use the default host and port for the Redis database which is `localhost:6379`.



```python
import redis
from redis.commands.search.indexDefinition import (
    IndexDefinition,
    IndexType
)
from redis.commands.search.query import Query
from redis.commands.search.field import (
    TagField,
    NumericField,
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

## Creating a Search Index in Redis

The below cells will show how to specify and create a search index in Redis. We will:

1. Set some constants for defining our index like the distance metric and the index name
2. Define the index schema with RediSearch fields
3. Create the index

```python
# Constants
INDEX_NAME = "product_embeddings"           # name of the search index
PREFIX = "doc"                            # prefix for the document keys
DISTANCE_METRIC = "L2"                # distance metric for the vectors (ex. COSINE, IP, L2)
NUMBER_OF_VECTORS = len(df)
```

```python
# Define RediSearch fields for each of the columns in the dataset
name = TextField(name="productDisplayName")
category = TagField(name="masterCategory")
articleType = TagField(name="articleType")
gender = TagField(name="gender")
season = TagField(name="season")
year = NumericField(name="year")
text_embedding = VectorField("product_vector",
    "FLAT", {
        "TYPE": "FLOAT32",
        "DIM": 1536,
        "DISTANCE_METRIC": DISTANCE_METRIC,
        "INITIAL_CAP": NUMBER_OF_VECTORS,
    }
)
fields = [name, category, articleType, gender, season, year, text_embedding]
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

## Generate OpenAI Embeddings and Load Documents into the Index

Now that we have a search index, we can load documents into it. We will use the dataframe containing the styles dataset loaded previously. In Redis, either the HASH or JSON (if using RedisJSON in addition to RediSearch) data types can be used to store documents. We will use the HASH data type in this example. The cells below will show how to get OpenAI embeddings for the different products and load documents into the index.

```python
# Use OpenAI get_embeddings batch requests to speed up embedding creation
def embeddings_batch_request(documents: pd.DataFrame):
    records = documents.to_dict("records")
    print("Records to process: ", len(records))
    product_vectors = []
    docs = []
    batchsize = 1000

    for idx,doc in enumerate(records,start=1):
        # create byte vectors
        docs.append(doc["product_text"])
        if idx % batchsize == 0:
            product_vectors += get_embeddings(docs, EMBEDDING_MODEL)
            docs.clear()
            print("Vectors processed ", len(product_vectors), end='\r')
    product_vectors += get_embeddings(docs, EMBEDDING_MODEL)
    print("Vectors processed ", len(product_vectors), end='\r')
    return product_vectors
```

```python
def index_documents(client: redis.Redis, prefix: str, documents: pd.DataFrame):
    product_vectors = embeddings_batch_request(documents)
    records = documents.to_dict("records")
    batchsize = 500

    # Use Redis pipelines to batch calls and save on round trip network communication
    pipe = client.pipeline()
    for idx,doc in enumerate(records,start=1):
        key = f"{prefix}:{str(doc['product_id'])}"

        # create byte vectors
        text_embedding = np.array((product_vectors[idx-1]), dtype=np.float32).tobytes()

        # replace list of floats with byte vectors
        doc["product_vector"] = text_embedding

        pipe.hset(key, mapping = doc)
        if idx % batchsize == 0:
            pipe.execute()
    pipe.execute()
```

```python
%%time
index_documents(redis_client, PREFIX, df)
print(f"Loaded {redis_client.info()['db0']['keys']} documents in Redis search index with name: {INDEX_NAME}")
```

```text
Records to process:  1978
Loaded 1978 documents in Redis search index with name: product_embeddings
CPU times: user 619 ms, sys: 78.9 ms, total: 698 ms
Wall time: 3.34 s
```

## Simple Vector Search Queries with OpenAI Query Embeddings

Now that we have a search index and documents loaded into it, we can run search queries. Below we will provide a function that will run a search query and return the results. Using this function we run a few queries that will show how you can utilize Redis as a vector database.

```python
def search_redis(
    redis_client: redis.Redis,
    user_query: str,
    index_name: str = "product_embeddings",
    vector_field: str = "product_vector",
    return_fields: list = ["productDisplayName", "masterCategory", "gender", "season", "year", "vector_score"],
    hybrid_fields = "*",
    k: int = 20,
    print_results: bool = True,
) -> List[dict]:

    # Use OpenAI to create embedding vector from user query
    embedded_query = openai.Embedding.create(input=user_query,
                                            model="text-embedding-3-small",
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
    if print_results:
        for i, product in enumerate(results.docs):
            score = 1 - float(product.vector_score)
            print(f"{i}. {product.productDisplayName} (Score: {round(score ,3) })")
    return results.docs
```

```python
# Execute a simple vector search in Redis
results = search_redis(redis_client, 'man blue jeans', k=10)
```

```text
0. John Players Men Blue Jeans (Score: 0.791)
1. Lee Men Tino Blue Jeans (Score: 0.775)
2. Peter England Men Party Blue Jeans (Score: 0.763)
3. Lee Men Blue Chicago Fit Jeans (Score: 0.761)
4. Lee Men Blue Chicago Fit Jeans (Score: 0.761)
5. French Connection Men Blue Jeans (Score: 0.74)
6. Locomotive Men Washed Blue Jeans (Score: 0.739)
7. Locomotive Men Washed Blue Jeans (Score: 0.739)
8. Do U Speak Green Men Blue Shorts (Score: 0.736)
9. Palm Tree Kids Boy Washed Blue Jeans (Score: 0.732)
```

## Hybrid Queries with Redis

The previous examples showed how run vector search queries with RediSearch. In this section, we will show how to combine vector search with other RediSearch fields for hybrid search. In the example below, we will combine vector search with full text search.

```python
# improve search quality by adding hybrid query for "man blue jeans" in the product vector combined with a phrase search for "blue jeans"
results = search_redis(redis_client,
                       "man blue jeans",
                       vector_field="product_vector",
                       k=10,
                       hybrid_fields='@productDisplayName:"blue jeans"'
                       )
```

```text
0. John Players Men Blue Jeans (Score: 0.791)
1. Lee Men Tino Blue Jeans (Score: 0.775)
2. Peter England Men Party Blue Jeans (Score: 0.763)
3. French Connection Men Blue Jeans (Score: 0.74)
4. Locomotive Men Washed Blue Jeans (Score: 0.739)
5. Locomotive Men Washed Blue Jeans (Score: 0.739)
6. Palm Tree Kids Boy Washed Blue Jeans (Score: 0.732)
7. Denizen Women Blue Jeans (Score: 0.725)
8. Jealous 21 Women Washed Blue Jeans (Score: 0.713)
9. Jealous 21 Women Washed Blue Jeans (Score: 0.713)
```

```python
# hybrid query for shirt in the product vector and only include results with the phrase "slim fit" in the title
results = search_redis(redis_client,
                       "shirt",
                       vector_field="product_vector",
                       k=10,
                       hybrid_fields='@productDisplayName:"slim fit"'
                       )
```

```text
0. Basics Men White Slim Fit Striped Shirt (Score: 0.633)
1. ADIDAS Men's Slim Fit White T-shirt (Score: 0.628)
2. Basics Men Blue Slim Fit Checked Shirt (Score: 0.627)
3. Basics Men Blue Slim Fit Checked Shirt (Score: 0.627)
4. Basics Men Red Slim Fit Checked Shirt (Score: 0.623)
5. Basics Men Navy Slim Fit Checked Shirt (Score: 0.613)
6. Lee Rinse Navy Blue Slim Fit Jeans (Score: 0.558)
7. Tokyo Talkies Women Navy Slim Fit Jeans (Score: 0.552)
```

```python
# hybrid query for watch in the product vector and only include results with the tag "Accessories" in the masterCategory field
results = search_redis(redis_client,
                       "watch",
                       vector_field="product_vector",
                       k=10,
                       hybrid_fields='@masterCategory:{Accessories}'
                       )
```

```text
0. Titan Women Gold Watch (Score: 0.544)
1. Being Human Men Grey Dial Blue Strap Watch (Score: 0.544)
2. Police Men Black Dial Watch PL12170JSB (Score: 0.544)
3. Titan Men Black Watch (Score: 0.543)
4. Police Men Black Dial Chronograph Watch PL12777JS-02M (Score: 0.542)
5. CASIO Youth Series Digital Men Black Small Dial Digital Watch W-210-1CVDF I065 (Score: 0.542)
6. Titan Women Silver Watch (Score: 0.542)
7. Police Men Black Dial Watch PL12778MSU-61 (Score: 0.541)
8. Titan Raga Women Gold Watch (Score: 0.539)
9. ADIDAS Original Men Black Dial Chronograph Watch ADH2641 (Score: 0.539)
```

```python
# hybrid query for sandals in the product vector and only include results within the 2011-2012 year range
results = search_redis(redis_client,
                       "sandals",
                       vector_field="product_vector",
                       k=10,
                       hybrid_fields='@year:[2011 2012]'
                       )
```

```text
0. Enroute Teens Orange Sandals (Score: 0.701)
1. Fila Men Camper Brown Sandals (Score: 0.692)
2. Clarks Men Black Leather Closed Sandals (Score: 0.691)
3. Coolers Men Black Sandals (Score: 0.69)
4. Coolers Men Black Sandals (Score: 0.69)
5. Enroute Teens Brown Sandals (Score: 0.69)
6. Crocs Dora Boots Pink Sandals (Score: 0.69)
7. Enroute Men Leather Black Sandals (Score: 0.685)
8. ADIDAS Men Navy Blue Benton Sandals (Score: 0.684)
9. Coolers Men Black Sports Sandals (Score: 0.684)
```

```python
# hybrid query for sandals in the product vector and only include results within the 2011-2012 year range from the summer season
results = search_redis(redis_client,
                       "blue sandals",
                       vector_field="product_vector",
                       k=10,
                       hybrid_fields='(@year:[2011 2012] @season:{Summer})'
                       )
```

```text
0. ADIDAS Men Navy Blue Benton Sandals (Score: 0.691)
1. Enroute Teens Brown Sandals (Score: 0.681)
2. ADIDAS Women's Adi Groove Blue Flip Flop (Score: 0.672)
3. Enroute Women Turquoise Blue Flats (Score: 0.671)
4. Red Tape Men Black Sandals (Score: 0.67)
5. Enroute Teens Orange Sandals (Score: 0.661)
6. Vans Men Blue Era Scilla Plaid Shoes (Score: 0.658)
7. FILA Men Aruba Navy Blue Sandal (Score: 0.657)
8. Quiksilver Men Blue Flip Flops (Score: 0.656)
9. Reebok Men Navy Twist Sandals (Score: 0.656)
```

```python
# hybrid query for a brown belt filtering results by a year (NUMERIC) with a specific article types (TAG) and with a brand name (TEXT)
results = search_redis(redis_client,
                       "brown belt",
                       vector_field="product_vector",
                       k=10,
                       hybrid_fields='(@year:[2012 2012] @articleType:{Shirts | Belts} @productDisplayName:"Wrangler")'
                       )
```

```text
0. Wrangler Men Leather Brown Belt (Score: 0.67)
1. Wrangler Women Black Belt (Score: 0.639)
2. Wrangler Men Green Striped Shirt (Score: 0.575)
3. Wrangler Men Purple Striped Shirt (Score: 0.549)
4. Wrangler Men Griffith White Shirt (Score: 0.543)
5. Wrangler Women Stella Green Shirt (Score: 0.542)
```