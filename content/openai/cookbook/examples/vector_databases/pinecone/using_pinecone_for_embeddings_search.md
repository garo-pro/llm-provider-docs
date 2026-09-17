# Using Pinecone for Embeddings Search

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
- **Pinecone**
    - *Setup*: Here we'll set up the Python client for Pinecone. For more details go [here](https://docs.pinecone.io/docs/quickstart)
    - *Index Data*: We'll create an index with namespaces for __titles__ and __content__
    - *Search Data*: We'll test out both namespaces with search queries to confirm it works

Once you've run through this notebook you should have a basic understanding of how to setup and use vector databases, and can move on to more complex use cases making use of our embeddings.

## Setup

Import the required libraries and set the embedding model that we'd like to use.

```python
# We'll need to install the Pinecone client
!pip install pinecone-client

#Install wget to pull zip file
!pip install wget
```

```text
Requirement already satisfied: pinecone-client in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (2.2.2)
Requirement already satisfied: requests>=2.19.0 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (2.31.0)
Requirement already satisfied: pyyaml>=5.4 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (6.0)
Requirement already satisfied: loguru>=0.5.0 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (0.7.0)
Requirement already satisfied: typing-extensions>=3.7.4 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (4.5.0)
Requirement already satisfied: dnspython>=2.0.0 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (2.3.0)
Requirement already satisfied: python-dateutil>=2.5.3 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (2.8.2)
Requirement already satisfied: urllib3>=1.21.1 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (1.26.16)
Requirement already satisfied: tqdm>=4.64.1 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (4.65.0)
Requirement already satisfied: numpy>=1.22.0 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from pinecone-client) (1.25.0)
Requirement already satisfied: six>=1.5 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from python-dateutil>=2.5.3->pinecone-client) (1.16.0)
Requirement already satisfied: charset-normalizer<4,>=2 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from requests>=2.19.0->pinecone-client) (3.1.0)
Requirement already satisfied: idna<4,>=2.5 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from requests>=2.19.0->pinecone-client) (3.4)
Requirement already satisfied: certifi>=2017.4.17 in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (from requests>=2.19.0->pinecone-client) (2023.5.7)
Requirement already satisfied: wget in /Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages (3.2)
```

```python
import openai

from typing import List, Iterator
import pandas as pd
import numpy as np
import os
import wget
from ast import literal_eval

# Pinecone's client library for Python
import pinecone

# I've set this to our new embeddings model, this can be changed to the embedding model of your choice
EMBEDDING_MODEL = "text-embedding-3-small"

# Ignore unclosed SSL socket warnings - optional in case you get these errors
import warnings

warnings.filterwarnings(action="ignore", message="unclosed", category=ResourceWarning)
warnings.filterwarnings("ignore", category=DeprecationWarning)
```

```text
/Users/colin.jarvis/Documents/dev/cookbook/openai-cookbook/vector_db/lib/python3.10/site-packages/pinecone/index.py:4: TqdmExperimentalWarning: Using `tqdm.autonotebook.tqdm` in notebook mode. Use `tqdm.tqdm` instead to force console mode (e.g. in jupyter console)
  from tqdm.autonotebook import tqdm
```

## Load data

In this section we'll load embedded data that we've prepared [in this article](https://developers.openai.com/cookbook/examples/Embedding_Wikipedia_articles_for_search.ipynb).

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

## Pinecone

The next option we'll look at is **Pinecone**, a managed vector database which offers a cloud-native option.

Before you proceed with this step you'll need to navigate to [Pinecone](https://www.pinecone.io), sign up and then save your API key as an environment variable titled ```PINECONE_API_KEY```.

For section we will:
- Create an index with multiple namespaces for article titles and content
- Store our data in the index with separate searchable "namespaces" for article **titles** and **content**
- Fire some similarity search queries to verify our setup is working

```python
api_key = os.getenv("PINECONE_API_KEY")
pinecone.init(api_key=api_key)
```

### Create Index

First we will need to create an index, which we'll call `wikipedia-articles`. Once we have an index, we can create multiple namespaces, which can make a single index searchable for various use cases. For more details, consult [Pinecone documentation](https://docs.pinecone.io/docs/namespaces#:~:text=Pinecone%20allows%20you%20to%20partition,different%20subsets%20of%20your%20index.).

If you want to batch insert to your index in parallel to increase insertion speed then there is a great guide in the Pinecone documentation on [batch inserts in parallel](https://docs.pinecone.io/docs/insert-data#sending-upserts-in-parallel).

```python
# Models a simple batch generator that make chunks out of an input DataFrame
class BatchGenerator:
    
    
    def __init__(self, batch_size: int = 10) -> None:
        self.batch_size = batch_size
    
    # Makes chunks out of an input DataFrame
    def to_batches(self, df: pd.DataFrame) -> Iterator[pd.DataFrame]:
        splits = self.splits_num(df.shape[0])
        if splits <= 1:
            yield df
        else:
            for chunk in np.array_split(df, splits):
                yield chunk

    # Determines how many chunks DataFrame contains
    def splits_num(self, elements: int) -> int:
        return round(elements / self.batch_size)
    
    __call__ = to_batches

df_batcher = BatchGenerator(300)
```

```python
# Pick a name for the new index
index_name = 'wikipedia-articles'

# Check whether the index with the same name already exists - if so, delete it
if index_name in pinecone.list_indexes():
    pinecone.delete_index(index_name)
    
# Creates new index
pinecone.create_index(name=index_name, dimension=len(article_df['content_vector'][0]))
index = pinecone.Index(index_name=index_name)

# Confirm our index was created
pinecone.list_indexes()
```

```text
['podcasts', 'wikipedia-articles']
```

```python
# Upsert content vectors in content namespace - this can take a few minutes
print("Uploading vectors to content namespace..")
for batch_df in df_batcher(article_df):
    index.upsert(vectors=zip(batch_df.vector_id, batch_df.content_vector), namespace='content')
```

```text
Uploading vectors to content namespace..
```

```python
# Upsert title vectors in title namespace - this can also take a few minutes
print("Uploading vectors to title namespace..")
for batch_df in df_batcher(article_df):
    index.upsert(vectors=zip(batch_df.vector_id, batch_df.title_vector), namespace='title')
```

```text
Uploading vectors to title namespace..
```

```python
# Check index size for each namespace to confirm all of our docs have loaded
index.describe_index_stats()
```

```text
{'dimension': 1536,
 'index_fullness': 0.1,
 'namespaces': {'content': {'vector_count': 25000},
                'title': {'vector_count': 25000}},
 'total_vector_count': 50000}
```

### Search data

Now we'll enter some dummy searches and check we get decent results back

```python
# First we'll create dictionaries mapping vector IDs to their outputs so we can retrieve the text for our search results
titles_mapped = dict(zip(article_df.vector_id,article_df.title))
content_mapped = dict(zip(article_df.vector_id,article_df.text))
```

```python
def query_article(query, namespace, top_k=5):
    '''Queries an article using its title in the specified
     namespace and prints results.'''

    # Create vector embeddings based on the title column
    embedded_query = openai.Embedding.create(
                                            input=query,
                                            model=EMBEDDING_MODEL,
                                            )["data"][0]['embedding']

    # Query namespace passed as parameter using title vector
    query_result = index.query(embedded_query, 
                                      namespace=namespace, 
                                      top_k=top_k)

    # Print query results 
    print(f'\nMost similar results to {query} in "{namespace}" namespace:\n')
    if not query_result.matches:
        print('no query result')
    
    matches = query_result.matches
    ids = [res.id for res in matches]
    scores = [res.score for res in matches]
    df = pd.DataFrame({'id':ids, 
                       'score':scores,
                       'title': [titles_mapped[_id] for _id in ids],
                       'content': [content_mapped[_id] for _id in ids],
                       })
    
    counter = 0
    for k,v in df.iterrows():
        counter += 1
        print(f'{v.title} (score = {v.score})')
    
    print('\n')

    return df
```

```python
query_output = query_article('modern art in Europe','title')
```

```text

Most similar results to modern art in Europe in "title" namespace:

Museum of Modern Art (score = 0.875177085)
Western Europe (score = 0.867441177)
Renaissance art (score = 0.864156306)
Pop art (score = 0.860346854)
Northern Europe (score = 0.854658186)
```

```python
content_query_output = query_article("Famous battles in Scottish history",'content')
```

```text

Most similar results to Famous battles in Scottish history in "content" namespace:

Battle of Bannockburn (score = 0.869336188)
Wars of Scottish Independence (score = 0.861470938)
1651 (score = 0.852588475)
First War of Scottish Independence (score = 0.84962213)
Robert I of Scotland (score = 0.846214116)
```