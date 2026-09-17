# Redis as a Context Store with OpenAI Chat

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook demonstrates how to use Redis as high-speed context memory with ChatGPT.

## Prerequisites
* Redis instance with the Redis Search and Redis JSON modules
* Redis-py client lib
* OpenAI Python client lib
* OpenAI API key

## Installation
Install Python modules necessary for the examples.

```python
! pip install -q redis openai python-dotenv 'openai[datalib]'
```

## OpenAI API Key
Create a .env file and add your OpenAI key to it

```python
OPENAI_API_KEY=your_key
```

## OpenAI Setup
Key load + helper function for chat completion

```python
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()
oai_client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def get_completion(prompt, model="gpt-3.5-turbo"):
    messages = [{"role": "user", "content": prompt}]
    response = oai_client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=0,
    )
    return response.choices[0].message.content
```

## Experiment - Chat Completion on a Topic outside of the Model's Knowledge Cutoff Date
Gpt-3.5-turbo was trained on data up to Sep 2021.  Let's ask it a question about something that is beyond that date.  In this case, the FTX/Sam Bankman-Fried scandal. We are using an old model here for demonstration. Newer models such as got-4o has later knowledge cutoffs (late 2023) and will work here as well.

```python
prompt = "Is Sam Bankman-Fried's company, FTX, considered a well-managed company?"
response = get_completion(prompt)
print(response)
```

```text
Yes, FTX is generally considered a well-managed company. Sam Bankman-Fried, the founder and CEO of FTX, has a strong track record in the cryptocurrency industry and has successfully grown the company into one of the leading cryptocurrency exchanges in the world. FTX has also received positive reviews for its user-friendly platform, innovative products, and strong customer service. Additionally, FTX has been proactive in regulatory compliance and has taken steps to ensure the security of its users' funds. Overall, FTX is seen as a well-managed company in the cryptocurrency space.
```

## Incomplete Information
An unfortunate behavior of these AI systems is the system will provide a confident-sounding response - even when the system is not confident with its result.  One way to mitigate this is prompt re-engineering, as seen below.

```python
prompt ="Is Sam Bankman-Fried's company, FTX, considered a well-managed company?  If you don't know for certain, say unknown."
response = get_completion(prompt)
print(response)
```

```text
FTX is generally considered a well-managed company. Sam Bankman-Fried, the founder and CEO, has a strong reputation in the cryptocurrency industry for his leadership and strategic vision. FTX has also experienced significant growth and success since its founding in 2017. However, without specific insider knowledge or data, it is ultimately unknown whether FTX is definitively considered a well-managed company.
```

## Additional Context
Another way to combat incomplete information is to give the system more information such that it can make intelligent decisions vs guessing.  We'll use Redis as the source for that additional context.  We'll pull in business news articles from after the GPT knowledge cut-off date such that the system will have a better understanding of how FTX was actually managed. 

## Start the Redis Stack Docker container

```python
! docker compose up -d
```

## Connect Redis client

```python
from redis import from_url

REDIS_URL = 'redis://localhost:6379'
client = from_url(REDIS_URL)
client.ping()
```

```text
True
```

## Create Index
[FT.CREATE](https://redis.io/commands/ft.create/)

```python
from redis.commands.search.field import TextField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

schema = [ VectorField('$.vector', 
            "FLAT", 
            {   "TYPE": 'FLOAT32', 
                "DIM": 1536, 
                "DISTANCE_METRIC": "COSINE"
            },  as_name='vector' ),
            TextField('$.content', as_name='content')
        ]
idx_def = IndexDefinition(index_type=IndexType.JSON, prefix=['doc:'])
try: 
    client.ft('idx').dropindex()
except:
    pass
client.ft('idx').create_index(schema, definition=idx_def)
```

```text
b'OK'
```

## Load Data Files into Redis as JSON Objects with Text and Vector Fields
[Redis JSON](https://redis.io/docs/stack/json/)

```python
directory = './assets/'
model = 'text-embedding-3-small'
i = 1

for file in os.listdir(directory):
    with open(os.path.join(directory, file), 'r') as f:
        content = f.read()
        # Create the embedding using the new client-based method
        response = oai_client.embeddings.create(
            model=model,
            input=[content]
        )
        # Access the embedding from the response object
        vector = response.data[0].embedding
        
        # Store the content and vector using your JSON client
        client.json().set(f'doc:{i}', '$', {'content': content, 'vector': vector})
    i += 1
```

## Embed the Question and Perform VSS to find the most relevant document
[KNN Search](https://redis.io/docs/stack/search/reference/vectors/#knn-search)

```python
from redis.commands.search.query import Query
import numpy as np

response = oai_client.embeddings.create(
    input=[prompt],
    model=model
)
# Extract the embedding vector from the response
embedding_vector = response.data[0].embedding

# Convert the embedding to a numpy array of type float32 and then to bytes
vec = np.array(embedding_vector, dtype=np.float32).tobytes()

# Build and execute the Redis query
q = Query('*=>[KNN 1 @vector $query_vec AS vector_score]') \
    .sort_by('vector_score') \
    .return_fields('content') \
    .dialect(2)
params = {"query_vec": vec}

context = client.ft('idx').search(q, query_params=params).docs[0].content
print(context)
```

## Repeat the Question to OpenAI with context
Now that we have relevant context, add that to the prompt to OpenAI and get a very different response.

````python
prompt = f"""
Using the information delimited by triple backticks, answer this question: Is Sam Bankman-Fried's company, FTX, considered a well-managed company?

Context: ```{context}```
"""

response = get_completion(prompt)
print(response)
````

```text
Based on the information provided, FTX, Sam Bankman-Fried's company, is not considered a well-managed company. The company has faced bankruptcy proceedings, mishandling of customer funds, unauthorized transactions, freezing of assets by regulatory authorities, and a lack of trustworthy financial information. The new CEO, John J. Ray III, described the situation as a "complete failure of corporate controls" and indicated gross mismanagement. Additionally, the company's financial situation, lack of record-keeping, and use of inadequate accounting tools despite handling billions of dollars have raised serious concerns about its management practices.
```