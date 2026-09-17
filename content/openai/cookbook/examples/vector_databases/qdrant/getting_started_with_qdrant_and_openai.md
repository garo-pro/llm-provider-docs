# Using Qdrant as a vector database for OpenAI embeddings

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook guides you step by step on using **`Qdrant`** as a vector database for OpenAI embeddings. [Qdrant](https://qdrant.tech) is a high-performant vector search database written in Rust. It offers RESTful and gRPC APIs to manage your embeddings. There is an official Python [qdrant-client](https://github.com/qdrant/qdrant_client) that eases the integration with your apps.

This notebook presents an end-to-end process of:
1. Using precomputed embeddings created by OpenAI API.
2. Storing the embeddings in a local instance of Qdrant.
3. Converting raw text query to an embedding with OpenAI API.
4. Using Qdrant to perform the nearest neighbour search in the created collection.

### What is Qdrant

[Qdrant](https://qdrant.tech) is an Open Source vector database that allows storing neural embeddings along with the metadata, a.k.a [payload](https://qdrant.tech/documentation/payload/). Payloads are not only available for keeping some additional attributes of a particular point, but might be also used for filtering. [Qdrant](https://qdrant.tech) offers a unique filtering mechanism which is built-in into the vector search phase, what makes it really efficient.

### Deployment options

[Qdrant](https://qdrant.tech) might be launched in various ways, depending on the target load on the application it might be hosted:

- Locally or on premise, with Docker containers
- On Kubernetes cluster, with the [Helm chart](https://github.com/qdrant/qdrant-helm)
- Using [Qdrant Cloud](https://cloud.qdrant.io/)

### Integration

[Qdrant](https://qdrant.tech) provides both RESTful and gRPC APIs which makes integration easy, no matter the programming language you use. However, there are some official clients for the most popular languages available, and if you use Python then the [Python Qdrant client library](https://github.com/qdrant/qdrant_client) might be the best choice.

## Prerequisites

For the purposes of this exercise we need to prepare a couple of things:

1. Qdrant server instance. In our case a local Docker container.
2. The [qdrant-client](https://github.com/qdrant/qdrant_client) library to interact with the vector database.
3. An [OpenAI API key](https://platform.openai.com/settings/organization/api-keys).

### Start Qdrant server

We're going to use a local Qdrant instance running in a Docker container. The easiest way to launch it is to use the attached [docker-compose.yaml] file and run the following command:

```python
! docker compose up -d
```

```text
[1A[1B[0G[?25l[+] Running 1/0
 [32m✔[0m Container qdrant-qdrant-1  [32mRunning[0m                                      [34m0.0s [0m
[?25h
```

We might validate if the server was launched successfully by running a simple curl command:

```python
! curl http://localhost:6333
```

```text
{"title":"qdrant - vector search engine","version":"1.3.0"}
```

### Install requirements

This notebook obviously requires the `openai` and `qdrant-client` packages, but there are also some other additional libraries we will use. The following command installs them all:


```python
! pip install openai qdrant-client pandas wget
```

### Prepare your OpenAI API key

The OpenAI API key is used for vectorization of the documents and queries.

If you don't have an OpenAI API key, you can get one from [https://beta.openai.com/account/api-keys](https://beta.openai.com/account/api-keys).

Once you get your key, please add it to your environment variables as `OPENAI_API_KEY` by running following command:

```python
! export OPENAI_API_KEY="your API key"
```

```python
# Test that your OpenAI API key is correctly set as an environment variable
# Note. if you run this notebook locally, you will need to reload your terminal and the notebook for the env variables to be live.
import os

# Note. alternatively you can set a temporary env variable like this:
# os.environ["OPENAI_API_KEY"] = "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

if os.getenv("OPENAI_API_KEY") is not None:
    print("OPENAI_API_KEY is ready")
else:
    print("OPENAI_API_KEY environment variable not found")
```

```text
OPENAI_API_KEY is ready
```

## Connect to Qdrant

Connecting to a running instance of Qdrant server is easy with the official Python library:

```python
import qdrant_client

client = qdrant_client.QdrantClient(
    host="localhost",
    prefer_grpc=True,
)
```

We can test the connection by running any available method:

```python
client.get_collections()
```

```text
CollectionsResponse(collections=[])
```

## Load data

In this section we are going to load the data prepared previous to this session, so you don't have to recompute the embeddings of Wikipedia articles with your own credits.

```python
import wget

embeddings_url = "https://cdn.openai.com/API/examples/data/vector_database_wikipedia_articles_embedded.zip"

# The file is ~700 MB so this will take some time
wget.download(embeddings_url)
```

```text
100% [......................................................................] 698933052 / 698933052
```

```text
'vector_database_wikipedia_articles_embedded (9).zip'
```

The downloaded file has to be then extracted:

```python
import zipfile

with zipfile.ZipFile("vector_database_wikipedia_articles_embedded.zip","r") as zip_ref:
    zip_ref.extractall("../data")
```

And we can finally load it from the provided CSV file:

```python
import pandas as pd

from ast import literal_eval

article_df = pd.read_csv('../data/vector_database_wikipedia_articles_embedded.csv')
# Read vectors from strings back into a list
article_df["title_vector"] = article_df.title_vector.apply(literal_eval)
article_df["content_vector"] = article_df.content_vector.apply(literal_eval)
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

## Index data

Qdrant stores data in __collections__ where each object is described by at least one vector and may contain an additional metadata called __payload__. Our collection will be called **Articles** and each object will be described by both **title** and **content** vectors. Qdrant does not require you to set up any kind of schema beforehand, so you can freely put points to the collection with a simple setup only.

We will start with creating a collection, and then we will fill it with our precomputed embeddings.

```python
from qdrant_client.http import models as rest

vector_size = len(article_df["content_vector"][0])

client.create_collection(
    collection_name="Articles",
    vectors_config={
        "title": rest.VectorParams(
            distance=rest.Distance.COSINE,
            size=vector_size,
        ),
        "content": rest.VectorParams(
            distance=rest.Distance.COSINE,
            size=vector_size,
        ),
    }
)
```

```text
True
```

```python
client.upsert(
    collection_name="Articles",
    points=[
        rest.PointStruct(
            id=k,
            vector={
                "title": v["title_vector"],
                "content": v["content_vector"],
            },
            payload=v.to_dict(),
        )
        for k, v in article_df.iterrows()
    ],
)
```

```text
UpdateResult(operation_id=0, status=<UpdateStatus.COMPLETED: 'completed'>)
```

```python
# Check the collection size to make sure all the points have been stored
client.count(collection_name="Articles")
```

```text
CountResult(count=25000)
```

## Search data

Once the data is put into Qdrant we will start querying the collection for the closest vectors. We may provide an additional parameter `vector_name` to switch from title to content based search. Since the precomputed embeddings were created with `text-embedding-ada-002` OpenAI model we also have to use it during search.


```python
from openai import OpenAI

openai_client = OpenAI()

def query_qdrant(query, collection_name, vector_name="title", top_k=20):
    # Creates embedding vector from user query
    embedded_query = openai_client.embeddings.create(
        input=query,
        model="text-embedding-ada-002",
    ).data[0].embedding

    query_results = client.search(
        collection_name=collection_name,
        query_vector=(
            vector_name, embedded_query
        ),
        limit=top_k,
    )

    return query_results
```

```python
query_results = query_qdrant("modern art in Europe", "Articles")
for i, article in enumerate(query_results):
    print(f"{i + 1}. {article.payload['title']} (Score: {round(article.score, 3)})")
```

```text
1. Museum of Modern Art (Score: 0.875)
2. Western Europe (Score: 0.867)
3. Renaissance art (Score: 0.864)
4. Pop art (Score: 0.86)
5. Northern Europe (Score: 0.855)
6. Hellenistic art (Score: 0.853)
7. Modernist literature (Score: 0.847)
8. Art film (Score: 0.843)
9. Central Europe (Score: 0.843)
10. European (Score: 0.841)
11. Art (Score: 0.841)
12. Byzantine art (Score: 0.841)
13. Postmodernism (Score: 0.84)
14. Eastern Europe (Score: 0.839)
15. Cubism (Score: 0.839)
16. Europe (Score: 0.839)
17. Impressionism (Score: 0.838)
18. Bauhaus (Score: 0.838)
19. Surrealism (Score: 0.837)
20. Expressionism (Score: 0.837)
```

```python
# This time we'll query using content vector
query_results = query_qdrant("Famous battles in Scottish history", "Articles", "content")
for i, article in enumerate(query_results):
    print(f"{i + 1}. {article.payload['title']} (Score: {round(article.score, 3)})")
```

```text
1. Battle of Bannockburn (Score: 0.869)
2. Wars of Scottish Independence (Score: 0.861)
3. 1651 (Score: 0.852)
4. First War of Scottish Independence (Score: 0.85)
5. Robert I of Scotland (Score: 0.846)
6. 841 (Score: 0.844)
7. 1716 (Score: 0.844)
8. 1314 (Score: 0.837)
9. 1263 (Score: 0.836)
10. William Wallace (Score: 0.835)
11. Stirling (Score: 0.831)
12. 1306 (Score: 0.831)
13. 1746 (Score: 0.83)
14. 1040s (Score: 0.828)
15. 1106 (Score: 0.827)
16. 1304 (Score: 0.826)
17. David II of Scotland (Score: 0.825)
18. Braveheart (Score: 0.824)
19. 1124 (Score: 0.824)
20. Second War of Scottish Independence (Score: 0.823)
```