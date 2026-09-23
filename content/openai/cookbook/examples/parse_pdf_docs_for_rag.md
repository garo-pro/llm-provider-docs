# Parsing PDF documents for RAG applications

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook shows how to leverage GPT-4o to turn rich PDF documents such as slide decks or exports from web pages into usable content for your RAG application.

This technique can be used if you have a lot of unstructured data containing valuable information that you want to be able to retrieve as part of your RAG pipeline.

For example, you could build a Knowledge Assistant that could answer user queries about your company or product based on information contained in PDF documents. 

The example documents used in this notebook are located at [data/example_pdfs](https://developers.openai.com/cookbook/examples/data/example_pdfs). They are related to OpenAI's APIs and various techniques that can be used as part of LLM projects.

## Data preparation

In this section, we will process our input data to prepare it for retrieval.

We will do this in 2 ways:

1. Extracting text with pdfminer
2. Converting the PDF pages to images to analyze them with GPT-4o

You can skip the 1st method if you want to only use the content inferred from the image analysis.

### Setup

We need to install a few libraries to convert the PDF to images and extract the text (optional).

**Note: You need to install `poppler` on your machine for the `pdf2image` library to work. You can follow the instructions to install it [here](https://pypi.org/project/pdf2image/).**

```python
%pip install pdf2image -q
%pip install pdfminer -q
%pip install pdfminer.six -q
%pip install openai -q
%pip install scikit-learn -q
%pip install rich -q
%pip install tqdm -q
%pip install pandas -q
```

```text
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
Note: you may need to restart the kernel to use updated packages.
```

```python
# Imports
from pdf2image import convert_from_path
from pdf2image.exceptions import (
    PDFInfoNotInstalledError,
    PDFPageCountError,
    PDFSyntaxError
)
from pdfminer.high_level import extract_text
import base64
import io
import os
import concurrent.futures
from tqdm import tqdm
from openai import OpenAI
import re
import pandas as pd 
from sklearn.metrics.pairwise import cosine_similarity
import json
import numpy as np
from rich import print
from ast import literal_eval
```

### File processing

```python
def convert_doc_to_images(path):
    images = convert_from_path(path)
    return images

def extract_text_from_doc(path):
    text = extract_text(path)
    return text
```

#### Testing with an example

```python
file_path = "data/example_pdfs/fine-tuning-deck.pdf"

images = convert_doc_to_images(file_path)
```

```python
text = extract_text_from_doc(file_path)
```

```python
for img in images:
    display(img)
```

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-0.png)

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-1.png)

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-2.png)

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-3.png)

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-4.png)

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-5.png)

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-10-output-6.png)

### Image analysis with GPT-4o

After converting a PDF file to multiple images, we'll use GPT-4o to analyze the content based on the images.

```python
# Initializing OpenAI client - see https://platform.openai.com/docs/quickstart?context=python
client = OpenAI()
```

```python
# Converting images to base64 encoded images in a data URI format to use with the ChatCompletions API
def get_img_uri(img):
    png_buffer = io.BytesIO()
    img.save(png_buffer, format="PNG")
    png_buffer.seek(0)

    base64_png = base64.b64encode(png_buffer.read()).decode('utf-8')

    data_uri = f"data:image/png;base64,{base64_png}"
    return data_uri
```

```python
system_prompt = '''
You will be provided with an image of a PDF page or a slide. Your goal is to deliver a detailed and engaging presentation about the content you see, using clear and accessible language suitable for a 101-level audience.

If there is an identifiable title, start by stating the title to provide context for your audience.

Describe visual elements in detail:

- **Diagrams**: Explain each component and how they interact. For example, "The process begins with X, which then leads to Y and results in Z."
  
- **Tables**: Break down the information logically. For instance, "Product A costs X dollars, while Product B is priced at Y dollars."

Focus on the content itself rather than the format:

- **DO NOT** include terms referring to the content format.
  
- **DO NOT** mention the content type. Instead, directly discuss the information presented.

Keep your explanation comprehensive yet concise:

- Be exhaustive in describing the content, as your audience cannot see the image.
  
- Exclude irrelevant details such as page numbers or the position of elements on the image.

Use clear and accessible language:

- Explain technical terms or concepts in simple language appropriate for a 101-level audience.

Engage with the content:

- Interpret and analyze the information where appropriate, offering insights to help the audience understand its significance.

------

If there is an identifiable title, present the output in the following format:

{TITLE}

{Content description}

If there is no clear title, simply provide the content description.
'''

def analyze_image(data_uri):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_prompt},
            {
                "role": "user",
                "content": [
                    {
                    "type": "image_url",
                    "image_url": {
                        "url": f"{data_uri}"
                    }
                    }
                ]
                },
        ],
        max_tokens=500,
        temperature=0,
        top_p=0.1
    )
    return response.choices[0].message.content
```

#### Testing with an example

```python
img = images[2]
display(img)
data_uri = get_img_uri(img)
```

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/parse_pdf_docs_for_rag/cell-16-output-0.png)

```python
res = analyze_image(data_uri)
print(res)
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What is Fine-tuning

Fine-tuning is a process where a pre-existing model, known as a public model, is trained using specific training 
data. This involves providing the model with a set of input/output examples to learn from. The goal is to adjust 
the model so that it can respond accurately to similar inputs in the future.

The diagram illustrates this process: starting with a public model, training data is used in a training phase to 
produce a fine-tuned model. This refined model is better equipped to handle specific tasks or datasets.

For effective fine-tuning, it is recommended to use between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples, although the minimum requirement is
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> examples. This ensures the model has enough data to learn from and improve its performance.
</pre>

#### Processing all documents

```python
files_path = "data/example_pdfs"

all_items = os.listdir(files_path)
files = [item for item in all_items if os.path.isfile(os.path.join(files_path, item))]
```

```python
def analyze_doc_image(img):
    img_uri = get_img_uri(img)
    data = analyze_image(img_uri)
    return data
```

We will list all files in the example folder and process them by 
1. Extracting the text
2. Converting the docs to images
3. Analyzing pages with GPT-4o

Note: This takes about ~2 mins to run. Feel free to skip and load directly the result file (see below).

```python
docs = []

for f in files:
    
    path = f"{files_path}/{f}"
    doc = {
        "filename": f
    }
    text = extract_text_from_doc(path)
    doc['text'] = text
    imgs = convert_doc_to_images(path)
    pages_description = []
    
    print(f"Analyzing pages for doc {f}")
    
    # Concurrent execution
    with concurrent.futures.ThreadPoolExecutor(max_workers=8) as executor:
        
        # Removing 1st slide as it's usually just an intro
        futures = [
            executor.submit(analyze_doc_image, img)
            for img in imgs[1:]
        ]
        
        with tqdm(total=len(imgs)-1) as pbar:
            for _ in concurrent.futures.as_completed(futures):
                pbar.update(1)
        
        for f in futures:
            res = f.result()
            pages_description.append(res)
        
    doc['pages_description'] = pages_description
    docs.append(doc)
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Analyzing pages for doc rag-deck.pdf
</pre>

```text
100%|██████████| 19/19 [00:20<00:00,  1.07s/it]
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Analyzing pages for doc models-page.pdf
</pre>

```text
100%|██████████| 9/9 [00:15<00:00,  1.76s/it]
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Analyzing pages for doc evals-decks.pdf
</pre>

```text
100%|██████████| 12/12 [00:12<00:00,  1.08s/it]
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Analyzing pages for doc fine-tuning-deck.pdf
</pre>

```text
100%|██████████| 6/6 [00:07<00:00,  1.31s/it]
```

```python
# Saving result to file for later
json_path = "data/parsed_pdf_docs.json"

with open(json_path, 'w') as f:
    json.dump(docs, f)
```

```python
# Optional: load content from the saved file
with open(json_path, 'r') as f:
    docs = json.load(f)
```

### Embedding content
Before embedding the content, we will chunk it logically by page.
For real-world scenarios, you could explore more advanced ways to chunk the content:
- Cutting it into smaller pieces
- Adding data - such as the slide title, deck title and/or the doc description - at the beginning of each piece of content. That way, each independent chunk can be in context

For the sake of brevity, we will use a very simple chunking strategy and rely on separators to split the text by page.

```python
# Chunking content by page and merging together slides text & description if applicable
content = []
for doc in docs:
    # Removing first slide as well
    text = doc['text'].split('\f')[1:]
    description = doc['pages_description']
    description_indexes = []
    for i in range(len(text)):
        slide_content = text[i] + '\n'
        # Trying to find matching slide description
        slide_title = text[i].split('\n')[0]
        for j in range(len(description)):
            description_title = description[j].split('\n')[0]
            if slide_title.lower() == description_title.lower():
                slide_content += description[j].replace(description_title, '')
                # Keeping track of the descriptions added
                description_indexes.append(j)
        # Adding the slide content + matching slide description to the content pieces
        content.append(slide_content) 
    # Adding the slides descriptions that weren't used
    for j in range(len(description)):
        if j not in description_indexes:
            content.append(description[j])
```

```python
for c in content:
    print(c)
    print("\n\n-------------------------------\n\n")
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Overview

Retrieval-Augmented Generation 
enhances the capabilities of language 
models by combining them with a 
retrieval system. This allows the model 
to leverage external knowledge sources 
to generate more accurate and 
contextually relevant responses.

Example use cases

- Provide answers with up-to-date 

information

- Generate contextual responses

What we’ll cover

● Technical patterns

● Best practices

● Common pitfalls

● Resources

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What is RAG

Retrieve information to Augment the model’s knowledge and Generate the output

“What is your 
return policy?”

ask

result

search

LLM

return information

Total refunds: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>% of value vouchers: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days
$<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> discount on next order: > <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

“You can get a full refund up 
to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days after the 
purchase, then up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days 
you would get a voucher for 
half the value of your order”

Knowledge 
Base <span style="color: #800080; text-decoration-color: #800080">/</span> External 
sources

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>




RAG stands for <span style="color: #008000; text-decoration-color: #008000">"Retrieve information to Augment the model’s knowledge and Generate the output."</span> This process 
involves using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> to enhance its responses by accessing external information sources.

Here's how it works:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **User Query**: A user asks a question, such as <span style="color: #008000; text-decoration-color: #008000">"What is your return policy?"</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **LLM Processing**: The language model receives the question and initiates a search for relevant information.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Information Retrieval**: The LLM accesses a knowledge base or external sources to find the necessary details. 
In this example, the information retrieved includes:
   - Total refunds available from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days.
   - <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>% value vouchers for returns between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days.
   - A $<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> discount on the next order for returns after <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Response Generation**: The LLM uses the retrieved information to generate a coherent response for the user. 
For instance, it might say, <span style="color: #008000; text-decoration-color: #008000">"You can get a full refund up to 14 days after the purchase, then up to 30 days you </span>
<span style="color: #008000; text-decoration-color: #008000">would get a voucher for half the value of your order."</span>

This method allows the model to provide accurate and up-to-date answers by leveraging external data sources.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">When to use RAG

Good for  ✅

Not good for  ❌

●

●

Introducing new information to the model 

●

Teaching the model a speciﬁc format, style, 

to update its knowledge

Reducing hallucinations by controlling 

content

<span style="color: #800080; text-decoration-color: #800080">/</span>!\ Hallucinations can still happen with RAG

or language
➔ Use ﬁne-tuning or custom models instead

●

Reducing token usage
➔ Consider ﬁne-tuning depending on the use 

case

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>




**Good for:**

- **Introducing new information to the model:** RAG <span style="font-weight: bold">(</span>Retrieval-Augmented Generation<span style="font-weight: bold">)</span> is effective for updating a 
model's knowledge by incorporating new data.

- **Reducing hallucinations by controlling content:** While RAG can help minimize hallucinations, it's important to
note that they can still occur.

**Not good for:**

- **Teaching the model a specific format, style, or language:** For these tasks, it's better to use fine-tuning or 
custom models.

- **Reducing token usage:** If token usage is a concern, consider fine-tuning based on the specific use case.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns

Data preparation

Input processing

Retrieval

Answer Generation

● Chunking

●

●

Embeddings

Augmenting 
content

●

Input 
augmentation

● NER

●

Search

● Context window

● Multi-step 
retrieval

● Optimisation

●

Safety checks

●

Embeddings

● Re-ranking

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation

chunk documents into multiple 
pieces for easier consumption

content

embeddings

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.876</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.145</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.179</span>…

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…

Augment content 
using LLMs

Ex: parse text only, ask gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> to rephrase &#x26; 
summarize each part, generate bullet points…

BEST PRACTICES

Pre-process content for LLM 
consumption: 
Add summary, headers for each 
part, etc.
+ curate relevant data sources

Knowledge 
Base

COMMON PITFALLS

➔ Having too much low-quality 

content

➔ Having too large documents

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">7</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation: chunking

Why chunking?

If your system doesn’t require 
entire documents to provide 
relevant answers, you can 
chunk them into multiple pieces 
for easier consumption <span style="font-weight: bold">(</span>reduced 
cost &#x26; latency<span style="font-weight: bold">)</span>.

Other approaches: graphs or 
map-reduce

Things to consider

●

Overlap:

○

○

Should chunks be independent or overlap one 
another?
If they overlap, by how much?

●

Size of chunks: 

○ What is the optimal chunk size for my use case?
○

Do I want to include a lot in the context window or 
just the minimum?

● Where to chunk:

○

○

Should I chunk every N tokens or use speciﬁc 
separators? 
Is there a logical way to split the context that would 
help the retrieval process?

● What to return:

○

○

Should I return chunks across multiple documents 
or top chunks within the same doc?
Should chunks be linked together with metadata to 
indicate common properties?

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation: embeddings

What to embed?

Depending on your use case 
you might not want just to 
embed the text in the 
documents but metadata as well 
- anything that will make it easier 
to surface this speciﬁc chunk or 
document when performing a 
search

Examples

Embedding Q&#x26;A posts in a forum
You might want to embed the title of the posts, 
the text of the original question and the content of 
the top answers.
Additionally, if the posts are tagged by topic or 
with keywords, you can embed those too.

Embedding product specs
In additional to embedding the text contained in 
documents describing the products, you might 
want to add metadata that you have on the 
product such as the color, size, etc. in your 
embeddings.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">9</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation: augmenting content

What does “Augmenting 
content” mean?

Augmenting content refers to 
modiﬁcations of the original content 
to make it more digestible for a 
system relying on RAG. The 
modiﬁcations could be a change in 
format, wording, or adding 
descriptive content such as 
summaries or keywords.

Example approaches

Make it a guide*
Reformat the content to look more like 
a step-by-step guide with clear 
headings and bullet-points, as this 
format is more easily understandable 
by an LLM.

Add descriptive metadata*
Consider adding keywords or text that 
users might search for when thinking 
of a speciﬁc product or service.

Multimodality
Leverage models 
such as Whisper or 
GPT-4V to 
transform audio or 
visual content into 
text.
For example, you 
can use GPT-4V to 
generate tags for 
images or to 
describe slides.

* GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can do this for you with the right prompt

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Input processing

Process input according to task

Q&#x26;A
HyDE:  Ask LLM to hypothetically answer the 
question &#x26; use the answer to search the KB

embeddings

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.876</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.145</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.179</span>…

Content search
Prompt LLM to rephrase input &#x26; optionally add 
more context

query

SELECT * from items…

DB search
NER:  Find relevant entities to be used for a 
keyword search or to construct a search query

keywords

red

summer

BEST PRACTICES

Consider how to transform the 
input to match content in the 
database
Consider using metadata to 
augment the user input

COMMON PITFALLS

➔ Comparing directly the input 
to the database without 
considering the task 
speciﬁcities 

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">11</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Input processing: input augmentation

What is input augmentation?

Example approaches

Augmenting the input means turning 
it into something diﬀerent, either 
rephrasing it, splitting it in several 
inputs or expanding it.
This helps boost performance as 
the LLM might understand better 
the user intent.

Query 
expansion*
Rephrase the 
query to be 
more 
descriptive

HyDE*
Hypothetically 
answer the 
question &#x26; use 
the answer to 
search the KB

Splitting a query in N*
When there is more than <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> question or 
intent in a user query, consider 
splitting it in several queries

Fallback
Consider 
implementing a 
ﬂow where the LLM 
can ask for 
clariﬁcation when 
there is not enough 
information in the 
original user query 
to get a result
<span style="font-weight: bold">(</span>Especially relevant 
with tool usage<span style="font-weight: bold">)</span>

* GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can do this for you with the right prompt

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">12</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Input processing: NER

Why use NER?

Using NER <span style="font-weight: bold">(</span>Named Entity 
Recognition<span style="font-weight: bold">)</span> allows to extract 
relevant entities from the input, that 
can then be used for more 
deterministic search queries. 
This can be useful when the scope 
is very constrained.

Example

Searching for movies
If you have a structured database containing 
metadata on movies, you can extract genre, 
actors or directors names, etc. from the user 
query and use this to search the database

Note: You can use exact values or embeddings after 
having extracted the relevant entities

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval

re-ranking

INPUT

embeddings

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.876</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.145</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.179</span>…

query

SELECT * from items…

keywords

red

summer

Semantic 
search

RESULTS

RESULTS

vector DB

relational <span style="color: #800080; text-decoration-color: #800080">/</span> 
nosql db

FINAL RESULT

Used to 
generate output

BEST PRACTICES

Use a combination of semantic 
search and deterministic queries 
where possible

+ Cache output where possible

COMMON PITFALLS

➔ The wrong elements could be 
compared when looking at 
text similarity, that is why 
re-ranking is important

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval: search

How to search?

Semantic search

Keyword search

Search query

There are many diﬀerent 
approaches to search depending on 
the use case and the existing 
system.

Using embeddings, you 
can perform semantic 
searches. You can 
compare embeddings 
with what is in your 
database and ﬁnd the 
most similar.

If you have extracted 
speciﬁc entities or 
keywords to search for, 
you can search for these 
in your database.

Based on the extracted 
entities you have or the 
user input as is, you can 
construct search queries 
<span style="font-weight: bold">(</span>SQL, cypher…<span style="font-weight: bold">)</span> and use 
these queries to search 
your database.

You can use a hybrid approach and combine several of these.
You can perform multiple searches in parallel or in sequence, or 
search for keywords with their embeddings for example.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">15</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval: multi-step retrieval

What is multi-step retrieval?

In some cases, there might be 
several actions to be performed to 
get the required information to 
generate an answer.

Things to consider

●

Framework to be used:

○ When there are multiple steps to perform, 
consider whether you want to handle this 
yourself or use a framework to make it easier

●

Cost &#x26; Latency:

○

○

Performing multiple steps at the retrieval 
stage can increase latency and cost 
signiﬁcantly
Consider performing actions in parallel to 
reduce latency

●

Chain of Thought:

○

○

Guide the assistant with the chain of thought 
approach: break down instructions into 
several steps, with clear guidelines on 
whether to continue, stop or do something 
else. 
This is more appropriate when tasks need to 
be performed sequentially - for example: “if 
this didn’t work, then do this”

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval: re-ranking

What is re-ranking?

Example approaches

Re-ranking means re-ordering the 
results of the retrieval process to 
surface more relevant results.
This is particularly important when 
doing semantic searches.

Rule-based re-ranking
You can use metadata to rank results by relevance. For 
example, you can look at the recency of the documents, at 
tags, speciﬁc keywords in the title, etc.

Re-ranking algorithms
There are several existing algorithms/approaches you can use 
based on your use case: BERT-based re-rankers, 
cross-encoder re-ranking, TF-IDF algorithms…

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">17</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation

FINAL RESULT

Piece of content 
retrieved

LLM

Prompt including 
the content

User sees the 
ﬁnal result

BEST PRACTICES

Evaluate performance after each 
experimentation to assess if it’s 
worth exploring other paths
+ Implement guardrails if applicable

COMMON PITFALLS

➔ Going for ﬁne-tuning without 
trying other approaches
➔ Not paying attention to the 
way the model is prompted

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">18</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation: context window

How to manage context?

Depending on your use case, there are 
several things to consider when 
including retrieved content into the 
context window to generate an answer. 

Things to consider

●

Context window max size:

○

○

There is a maximum size, so putting too 
much content is not ideal
In conversation use cases, the 
conversation will be part of the context 
as well and will add to that size

●

Cost &#x26; Latency vs Accuracy:

○ More context results in increased 

latency and additional costs since there 
will be more input tokens
Less context might also result in 
decreased accuracy

○

●

“Lost in the middle” problem:

○ When there is too much context, LLMs 
tend to forget the text “in the middle” of 
the content and might look over some 
important information.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">19</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation: optimisation

How to optimise?

There are a few diﬀerent 
methods to consider when 
optimising a RAG application.
Try them from left to right, and 
iterate with several of these 
approaches if needed.

Prompt Engineering

Few-shot examples

Fine-tuning

At each point of the 
process, experiment with 
diﬀerent prompts to get 
the expected input format 
or generate a relevant 
output.
Try guiding the model if 
the process to get to the 
ﬁnal outcome contains 
several steps.

If the model doesn’t 
behave as expected, 
provide examples of what 
you want e.g. provide 
example user inputs and 
the expected processing 
format.

If giving a few examples 
isn’t enough, consider 
ﬁne-tuning a model with 
more examples for each 
step of the process: you 
can ﬁne-tune to get a 
speciﬁc input processing 
or output format.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">20</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation: safety checks

Why include safety checks?

Just because you provide the model 
with <span style="font-weight: bold">(</span>supposedly<span style="font-weight: bold">)</span> relevant context 
doesn’t mean the answer will 
systematically be truthful or on-point.
Depending on the use case, you 
might want to double-check. 

Example evaluation framework: RAGAS

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Overview**

Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span> enhances language models by integrating them with a retrieval system. This 
combination allows the model to access external knowledge sources, resulting in more accurate and contextually 
relevant responses. 

**Example Use Cases:**
- Providing answers with up-to-date information
- Generating contextual responses

**What We’ll Cover:**
- Technical patterns
- Best practices
- Common pitfalls
- Resources
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns**

This image outlines four key technical patterns involved in data processing and answer generation:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Data Preparation**
   - **Chunking**: Breaking down data into smaller, manageable pieces.
   - **Embeddings**: Converting data into numerical formats that can be easily processed by machine learning 
models.
   - **Augmenting Content**: Enhancing data with additional information to improve its quality or usefulness.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Input Processing**
   - **Input Augmentation**: Adding extra data or features to the input to improve model performance.
   - **NER <span style="font-weight: bold">(</span>Named Entity Recognition<span style="font-weight: bold">)</span>**: Identifying and classifying key entities in the text, such as names, 
dates, and locations.
   - **Embeddings**: Similar to data preparation, embeddings are used here to represent input data in a format 
suitable for processing.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Retrieval**
   - **Search**: Locating relevant information from a dataset.
   - **Multi-step Retrieval**: Using multiple steps or methods to refine the search process and improve accuracy.
   - **Re-ranking**: Adjusting the order of retrieved results based on relevance or other criteria.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Answer Generation**
   - **Context Window**: Using a specific portion of data to generate relevant answers.
   - **Optimisation**: Improving the efficiency and accuracy of the answer generation process.
   - **Safety Checks**: Ensuring that the generated answers are safe and appropriate for use.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Data Preparation**

This presentation focuses on the process of preparing data for easier consumption by large language models <span style="font-weight: bold">(</span>LLMs<span style="font-weight: bold">)</span>. 

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Content Chunking**: 
   - Documents are divided into smaller, manageable pieces. This makes it easier for LLMs to process the 
information.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Embeddings**:
   - Each chunk of content is converted into embeddings, which are numerical representations <span style="font-weight: bold">(</span>e.g., <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span><span style="font-weight: bold">)</span> that capture the semantic meaning of the text. These embeddings are then stored in a knowledge base.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Augmenting Content**:
   - Content can be enhanced using LLMs. For example, GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can be used to rephrase, summarize, and generate bullet
points from the text.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Best Practices**:
   - Pre-process content for LLM consumption by adding summaries and headers for each part.
   - Curate relevant data sources to ensure quality and relevance.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>. **Common Pitfalls**:
   - Avoid having too much low-quality content.
   - Ensure documents are not too large, as this can hinder processing efficiency.

This approach helps in organizing and optimizing data for better performance and understanding by LLMs.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Data Preparation - Chunking**

**Why Chunking?**

Chunking is a technique used when your system doesn't need entire documents to provide relevant answers. By 
breaking documents into smaller pieces, you can make data easier to process, which reduces cost and latency. This 
approach is beneficial for systems that need to handle large volumes of data efficiently. Other methods for data 
preparation include using graphs or map-reduce.

**Things to Consider**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Overlap:**
   - Should chunks be independent or overlap with one another?
   - If they overlap, by how much should they do so?

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Size of Chunks:**
   - What is the optimal chunk size for your specific use case?
   - Do you want to include a lot of information in the context window, or just the minimum necessary?

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Where to Chunk:**
   - Should you chunk every N tokens or use specific separators?
   - Is there a logical way to split the context that would aid the retrieval process?

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **What to Return:**
   - Should you return chunks across multiple documents or focus on top chunks within the same document?
   - Should chunks be linked together with metadata to indicate common properties?

These considerations help in designing an efficient chunking strategy that aligns with your system's requirements 
and goals.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"># Technical Patterns: Data Preparation - Embeddings

## What to Embed?

When preparing data for embedding, it's important to consider not just the text but also the metadata. This 
approach can enhance the searchability and relevance of the data. Here are some examples:

### Examples

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Embedding Q&#x26;A Posts in a Forum**
   - You might want to include the title of the posts, the original question, and the top answers.
   - Additionally, if the posts are tagged by topic or keywords, these can be embedded as well.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Embedding Product Specs**
   - Besides embedding the text from product descriptions, you can add metadata such as color, size, and other 
specifications to your embeddings.

By embedding both text and metadata, you can improve the ability to surface specific chunks or documents during a 
search.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Data Preparation - Augmenting Content**

**What does “Augmenting content” mean?**

Augmenting content involves modifying the original material to make it more accessible and understandable for 
systems that rely on Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span>. These modifications can include changes in format, 
wording, or the addition of descriptive elements like summaries or keywords.

**Example Approaches:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Make it a Guide:**
   - Reformat the content into a step-by-step guide with clear headings and bullet points. This structure is more 
easily understood by a Language Learning Model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span>. GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can assist with this transformation using the right 
prompts.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Add Descriptive Metadata:**
   - Incorporate keywords or text that users might search for when considering a specific product or service. This 
helps in making the content more searchable and relevant.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Multimodality:**
   - Utilize models like Whisper or GPT-4V to convert audio or visual content into text. For instance, GPT-4V can 
generate tags for images or describe slides, enhancing the content's accessibility and utility.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Input Processing**

This slide discusses methods for processing input data according to specific tasks, focusing on three main areas: 
Q&#x26;A, content search, and database <span style="font-weight: bold">(</span>DB<span style="font-weight: bold">)</span> search.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Q&#x26;A**: 
   - Uses a technique called HyDE, where a large language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> is asked to hypothetically answer a question.
This answer is then used to search the knowledge base <span style="font-weight: bold">(</span>KB<span style="font-weight: bold">)</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Content Search**:
   - Involves prompting the LLM to rephrase the input and optionally add more context to improve search results.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **DB Search**:
   - Utilizes Named Entity Recognition <span style="font-weight: bold">(</span>NER<span style="font-weight: bold">)</span> to find relevant entities. These entities are then used for keyword 
searches or to construct a search query.

The slide also highlights different output formats:
- **Embeddings**: Numerical representations of data, such as vectors <span style="font-weight: bold">(</span>e.g., <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span><span style="font-weight: bold">)</span>.
- **Query**: SQL-like statements for database searches <span style="font-weight: bold">(</span>e.g., SELECT * from items<span style="font-weight: bold">)</span>.
- **Keywords**: Specific terms extracted from the input <span style="font-weight: bold">(</span>e.g., <span style="color: #008000; text-decoration-color: #008000">"red,"</span> <span style="color: #008000; text-decoration-color: #008000">"summer"</span><span style="font-weight: bold">)</span>.

**Best Practices**:
- Transform the input to match the content in the database.
- Use metadata to enhance user input.

**Common Pitfalls**:
- Avoid directly comparing input to the database without considering the specific requirements of the task.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Input Processing - Input Augmentation**

**What is input augmentation?**

Input augmentation involves transforming the input into something different, such as rephrasing it, splitting it 
into several inputs, or expanding it. This process enhances performance by helping the language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> better 
understand the user's intent.

**Example Approaches:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Query Expansion**
   - Rephrase the query to make it more descriptive. This helps the LLM grasp the context and details more 
effectively.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **HyDE**
   - Hypothetically answer the question and use that answer to search the knowledge base <span style="font-weight: bold">(</span>KB<span style="font-weight: bold">)</span>. This approach can 
provide more relevant results by anticipating possible answers.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Splitting a Query in N**
   - When a user query contains multiple questions or intents, consider dividing it into several queries. This 
ensures each part is addressed thoroughly.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Fallback**
   - Implement a flow where the LLM can ask for clarification if the original query lacks sufficient information. 
This is particularly useful when using tools that require precise input.

*Note: GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can perform these tasks with the appropriate prompt.*
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Input Processing - NER

**Why use NER?**

Named Entity Recognition <span style="font-weight: bold">(</span>NER<span style="font-weight: bold">)</span> is a technique used to extract relevant entities from input data. This process is 
beneficial for creating more deterministic search queries, especially when the scope is very constrained. By 
identifying specific entities, such as names, dates, or locations, NER helps in refining and improving the accuracy
of searches.

**Example: Searching for Movies**

Consider a structured database containing metadata on movies. By using NER, you can extract specific entities like 
genre, actors, or directors' names from a user's query. This information can then be used to search the database 
more effectively. 

**Note:** After extracting the relevant entities, you can use exact values or embeddings to enhance the search 
process.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Retrieval

This diagram illustrates a retrieval process using technical patterns. The process begins with three types of 
input: embeddings, queries, and keywords.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Embeddings**: These are numerical representations <span style="font-weight: bold">(</span>e.g., <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span><span style="font-weight: bold">)</span> used for semantic search. They 
are processed through a vector database <span style="font-weight: bold">(</span>vector DB<span style="font-weight: bold">)</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Query**: This involves structured queries <span style="font-weight: bold">(</span>e.g., <span style="color: #008000; text-decoration-color: #008000">"SELECT * from items..."</span><span style="font-weight: bold">)</span> that interact with a relational or 
NoSQL database.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Keywords**: Simple search terms like <span style="color: #008000; text-decoration-color: #008000">"red"</span> and <span style="color: #008000; text-decoration-color: #008000">"summer"</span> are also used with the relational or NoSQL database.

The results from both the vector and relational/NoSQL databases are combined. The initial results undergo a 
re-ranking process to ensure accuracy and relevance, leading to the final result, which is then used to generate 
output.

**Best Practices**:
- Combine semantic search with deterministic queries for more effective retrieval.
- Cache outputs where possible to improve efficiency.

**Common Pitfalls**:
- Incorrect element comparison during text similarity checks can occur, highlighting the importance of re-ranking 
to ensure accurate results.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Retrieval - Search

**How to search?**

There are various approaches to searching, which depend on the use case and the existing system. Here are three 
main methods:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Semantic Search**:
   - This method uses embeddings to perform searches. 
   - By comparing embeddings with the data in your database, you can find the most similar matches.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Keyword Search**:
   - If you have specific entities or keywords extracted, you can search for these directly in your database.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Search Query**:
   - Based on extracted entities or direct user input, you can construct search queries <span style="font-weight: bold">(</span>such as SQL or Cypher<span style="font-weight: bold">)</span> to 
search your database.

Additionally, you can use a hybrid approach by combining several methods. This can involve performing multiple 
searches in parallel or in sequence, or searching for keywords along with their embeddings.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Retrieval - Multi-step Retrieval**

**What is multi-step retrieval?**

Multi-step retrieval involves performing several actions to obtain the necessary information to generate an answer.
This approach is useful when a single step is insufficient to gather all required data.

**Things to Consider**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Framework to be Used:**
   - When multiple steps are needed, decide whether to manage this process yourself or use a framework to simplify 
the task.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Cost &#x26; Latency:**
   - Performing multiple steps can significantly increase both latency and cost.
   - To mitigate latency, consider executing actions in parallel.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Chain of Thought:**
   - Use a chain of thought approach to guide the process. Break down instructions into clear steps, providing 
guidelines on whether to continue, stop, or take alternative actions.
   - This method is particularly useful for tasks that must be performed sequentially, such as <span style="color: #008000; text-decoration-color: #008000">"if this didn’t </span>
<span style="color: #008000; text-decoration-color: #008000">work, then do this."</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Retrieval - Re-ranking**

**What is re-ranking?**

Re-ranking involves re-ordering the results of a retrieval process to highlight more relevant outcomes. This is 
especially crucial in semantic searches, where understanding the context and meaning of queries is important.

**Example Approaches**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Rule-based Re-ranking**
   - This approach uses metadata to rank results by relevance. For instance, you might consider the recency of 
documents, tags, or specific keywords in the title to determine their importance.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Re-ranking Algorithms**
   - There are various algorithms available for re-ranking based on specific use cases. Examples include BERT-based
re-rankers, cross-encoder re-ranking, and TF-IDF algorithms. These methods apply different techniques to assess and
order the relevance of search results.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Answer Generation**

This diagram illustrates the process of generating answers using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span>. Here's a breakdown of the 
components and concepts:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Process Flow:**
   - A piece of content is retrieved and used to create a prompt.
   - This prompt is fed into the LLM, which processes it to generate a final result.
   - The user then sees this final result.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Best Practices:**
   - It's important to evaluate performance after each experiment. This helps determine if exploring other methods 
is beneficial.
   - Implementing guardrails can be useful to ensure the model's outputs are safe and reliable.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Common Pitfalls:**
   - Avoid jumping straight to fine-tuning the model without considering other approaches that might be more 
effective or efficient.
   - Pay close attention to how the model is prompted, as this can significantly impact the quality of the output.

By following these guidelines, you can optimize the use of LLMs for generating accurate and useful answers.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"># Technical Patterns: Answer Generation - Context Window

## How to Manage Context?

When generating answers using a context window, it's important to consider several factors based on your specific 
use case. Here are key points to keep in mind:

### Things to Consider

- **Context Window Max Size:**
  - The context window has a maximum size, so overloading it with too much content is not ideal.
  - In conversational scenarios, the conversation itself becomes part of the context, contributing to the overall 
size.

- **Cost &#x26; Latency vs. Accuracy:**
  - Including more context can lead to increased latency and higher costs due to the additional input tokens 
required.
  - Conversely, using less context might reduce accuracy.

- **<span style="color: #008000; text-decoration-color: #008000">"Lost in the Middle"</span> Problem:**
  - When the context is too extensive, language models may overlook or forget information that is <span style="color: #008000; text-decoration-color: #008000">"in the middle"</span> 
of the content, potentially missing important details.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Answer Generation Optimisation**

**How to optimise?**

When optimising a Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span> application, there are several methods to consider. These 
methods should be tried sequentially from left to right, and multiple approaches can be iterated if necessary.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Prompt Engineering**
   - Experiment with different prompts at each stage of the process to achieve the desired input format or generate
relevant output.
   - Guide the model through multiple steps to reach the final outcome.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Few-shot Examples**
   - If the model's behavior is not as expected, provide examples of the desired outcome.
   - Include sample user inputs and the expected processing format to guide the model.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Fine-tuning**
   - If a few examples are insufficient, consider fine-tuning the model with more examples for each process step.
   - Fine-tuning can help achieve a specific input processing or output format.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Answer Generation - Safety Checks

**Why include safety checks?**

Safety checks are crucial because providing a model with supposedly relevant context does not guarantee that the 
generated answer will be truthful or accurate. Depending on the use case, it is important to double-check the 
information to ensure reliability.

**RAGAS Score Evaluation Framework**

The RAGAS score is an evaluation framework that assesses both the generation and retrieval aspects of answer 
generation:

- **Generation:**
  - **Faithfulness:** This measures how factually accurate the generated answer is.
  - **Answer Relevancy:** This evaluates how relevant the generated answer is to the question.

- **Retrieval:**
  - **Context Precision:** This assesses the signal-to-noise ratio of the retrieved context, ensuring that the 
information is precise.
  - **Context Recall:** This checks if all relevant information required to answer the question is retrieved.

By using this framework, one can systematically evaluate and improve the quality of generated answers.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo ,  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> , and  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview  point to the latest model
version. You can verify this by looking at the response object after sending a request.
The response will include the specific model version used <span style="font-weight: bold">(</span>e.g.  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> <span style="font-weight: bold">)</span>.

We also offer static model versions that developers can continue using for at least
three months after an updated model has been introduced. With the new cadence of
model updates, we are also giving people the ability to contribute evals to help us

improve the model for different use cases. If you are interested, check out the OpenAI
Evals repository.

Learn more about model deprecation on our deprecation page.

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is a large multimodal model <span style="font-weight: bold">(</span>accepting text or image inputs and outputting text<span style="font-weight: bold">)</span>
that can solve difficult problems with greater accuracy than any of our previous

models, thanks to its broader general knowledge and advanced reasoning capabilities.

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is available in the OpenAI API to paying customers. Like  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo , GPT-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is optimized for chat but works well for traditional completions tasks using the Chat
Completions API. Learn how to use GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> in our text generation guide.

MODEL

DE S CRIPTION

CONTEXT
WIND OW

TRAINING
DATA

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview

New  GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>

Up to

Dec

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

The latest GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> model

tokens

intended to reduce cases of

“laziness” where the model
doesn’t complete a task.
Returns a maximum of

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
Learn more.

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview

Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview.

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-preview

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo model
featuring improved
instruction following, JSON

mode, reproducible outputs,
parallel function calling, and
more. Returns a maximum
of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens. This

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens

Up to
Dec
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens

Up to
Apr <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

MODEL

DE S CRIPTION

is a preview model.
Learn more.

CONTEXT
WIND OW

TRAINING
DATA

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> with the ability to
understand images, in

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens

Up to
Apr <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

addition to all other GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
Turbo capabilities. Currently
points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-

vision-preview.

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-vision-preview GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> with the ability to

understand images, in
addition to all other GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

Turbo capabilities. Returns a
maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output

tokens. This is a preview

model version. Learn more.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens

Up to
Apr <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>

Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span>

Up to

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. See

tokens

Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

continuous model upgrades.

Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> from

June 13th <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> with

improved function calling

support.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span>
tokens

Up to
Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k

Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>

32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. See

continuous model upgrades.
This model was never rolled
out widely in favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

Turbo.

Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k

from June 13th <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> with
improved function calling
support. This model was
never rolled out widely in

favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>

tokens

Up to

Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>

tokens

Up to

Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

For many basic tasks, the difference between GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> models is not
significant. However, in more complex reasoning situations, GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is much more
capable than any of our previous models.

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

Multilingual capabilities

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> outperforms both previous large language models and as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, most state-
of-the-art systems <span style="font-weight: bold">(</span>which often have benchmark-specific training or hand-
engineering<span style="font-weight: bold">)</span>. On the MMLU benchmark, an English-language suite of multiple-choice
questions covering <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">57</span> subjects, GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> not only outperforms existing models by a
considerable margin in English, but also demonstrates strong performance in other
languages.

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo models can understand and generate natural language or code and
have been optimized for chat using the Chat Completions API but work well for non-
chat tasks as well.

CONTEXT
WIND OW

TRAINING
DATA

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>

tokens

Up to Sep

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

MODEL

DE S CRIPTION

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>

New  Updated GPT <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo

The latest GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo
model with higher accuracy at

responding in requested

formats and a fix for a bug

which caused a text encoding
issue for non-English

language function calls.

Returns a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>

output tokens. Learn more.

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo

Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>

Up to Sep

turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. The gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-

tokens

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

turbo model alias will be

automatically upgraded from
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> to

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span> on

February 16th.

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo model with
improved instruction

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>
tokens

Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

following, JSON mode,
reproducible outputs, parallel
function calling, and more.
Returns a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>

output tokens. Learn more.

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

MODEL

DE S CRIPTION

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct Similar capabilities as GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>
era models. Compatible with
legacy Completions endpoint
and not Chat Completions.

CONTEXT
WIND OW

TRAINING
DATA

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
tokens

Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k

Legacy  Currently points to
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>
tokens

Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>

Legacy  Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-

turbo from June 13th <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.

Will be deprecated on June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>,
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
tokens

Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>

Legacy  Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>

Up to Sep

16k-turbo from June 13th

tokens

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Will be deprecated on

June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.

DALL·E

DALL·E is a AI system that can create realistic images and art from a description in

natural language. DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> currently supports the ability, given a prompt, to create a

new image with a specific size. DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> also support the ability to edit an existing

image, or create variations of a user provided image.

DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> is available through our Images API along with DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. You can try DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>

through ChatGPT Plus.

MODEL

DE S CRIPTION

dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>

New  DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>

The latest DALL·E model released in Nov <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Learn more.

dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> The previous DALL·E model released in Nov <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2022</span>. The 2nd iteration of
DALL·E with more realistic, accurate, and 4x greater resolution images
than the original model.

TTS

TTS is an AI model that converts text to natural sounding spoken text. We offer two
different model variates,  tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>  is optimized for real time text to speech use cases
and  tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd  is optimized for quality. These models can be used with the Speech

endpoint in the Audio API.

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

MODEL

DE S CRIPTION

tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>

New  Text-to-speech <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>
The latest text to speech model, optimized for speed.

tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd

New  Text-to-speech <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> HD
The latest text to speech model, optimized for quality.

Whisper

Whisper is a general-purpose speech recognition model. It is trained on a large dataset
of diverse audio and is also a multi-task model that can perform multilingual speech
recognition as well as speech translation and language identification. The Whisper v2-

large model is currently available through our API with the  whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>  model name.

Currently, there is no difference between the open source version of Whisper and the

version available through our API. However, through our API, we offer an optimized
inference process which makes running Whisper through our API much faster than

doing it through other means. For more technical details on Whisper, you can read the

paper.

Embeddings

Embeddings are a numerical representation of text that can be used to measure the

relatedness between two pieces of text. Embeddings are useful for search, clustering,

recommendations, anomaly detection, and classification tasks. You can read more
about our latest embedding models in the announcement blog post.

MODEL

DE S CRIPTION

text-embedding-
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-large

New  Embedding V3 large
Most capable embedding model for both

english and non-english tasks

text-embedding-

New  Embedding V3 small

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small

Increased performance over 2nd generation ada
embedding model

text-embedding-
ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>

Most capable 2nd generation embedding
model, replacing <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span> first generation models

OUTP UT
DIMENSION

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">072</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>

Moderation

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

The Moderation models are designed to check whether content complies with
OpenAI's usage policies. The models provide classification capabilities that look for
content in the following categories: hate, hate/threatening, self-harm, sexual,
sexual/minors, violence, and violence/graphic. You can find out more in our moderation

guide.

Moderation models take in an arbitrary sized input that is automatically broken up into
chunks of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens. In cases where the input is more than <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens,

truncation is used which in a rare condition may omit a small number of tokens from
the moderation check.

The final results from each request to the moderation endpoint shows the maximum

value on a per category basis. For example, if one chunk of 4K tokens had a category
score of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span> and the other had a score of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1901</span>, the results would show <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span> in the
API response since it is higher.

MODEL

DE S CRIPTION

MAX
TOKENS

text-moderation-latest Currently points to text-moderation-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span>.

text-moderation-stable Currently points to text-moderation-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span>.

text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span>

Most capable moderation model across
all categories.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>

GPT base

GPT base models can understand and generate natural language or code but are not
trained with instruction following. These models are made to be replacements for our

original GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> base models and use the legacy Completions API. Most customers

should use GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> or GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>.

MODEL

DE S CRIPTION

babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span> Replacement for the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> ada and

babbage base models.

davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span> Replacement for the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> curie and

davinci base models.

MAX
TOKENS

TRAINING
DATA

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span>
tokens

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span>
tokens

Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

How we use your data

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">7</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

Your data is your data.

As of March <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, data sent to the OpenAI API will not be used to train or improve

OpenAI models <span style="font-weight: bold">(</span>unless you explicitly opt in<span style="font-weight: bold">)</span>. One advantage to opting in is that the
models may get better at your use case over time.

To help identify abuse, API data may be retained for up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, after which it will be

deleted <span style="font-weight: bold">(</span>unless otherwise required by law<span style="font-weight: bold">)</span>. For trusted customers with sensitive
applications, zero data retention may be available. With zero data retention, request
and response bodies are not persisted to any logging mechanism and exist only in
memory in order to serve the request.

Note that this data policy does not apply to OpenAI's non-API consumer services like
ChatGPT or DALL·E Labs.

Default usage policies by endpoint

ENDP OINT

DATA USED
FOR TRAINING

DEFAULT
RETENTION

ELIGIBLE FOR
ZERO RETENTION

<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>*

No

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

Yes, except

image inputs*

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">files</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">threads</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">messages</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">runs</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/threads/runs/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">steps</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">generations</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">edits</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">variations</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>

No

No

No

No

No

No

No

No

No

No

<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span> No

Until deleted by

No

customer

Until deleted by

No

customer

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days *

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days *

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days *

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days *

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

Zero data
retention

No

No

No

No

No

No

No

Yes

-

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

Models - OpenAI API

ENDP OINT

DATA USED
FOR TRAINING

DEFAULT
RETENTION

ELIGIBLE FOR
ZERO RETENTION

<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>

No

<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>

No

No

No

No

Zero data
retention

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

Until deleted by
customer

Zero data
retention

-

No

No

-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days

Yes

* Image inputs via the  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview  model are not eligible for zero
retention.

* For the Assistants API, we are still evaluating the default retention period during the

Beta. We expect that the default retention period will be stable after the end of the

Beta.

For details, see our API data usage policies. To learn more about zero retention, get in

touch with our sales team.

Model endpoint compatibility

ENDP OINT

L ATE ST MODEL S

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>

All models except gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0301</span>

supported. The retrieval tool requires gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-

turbo-preview <span style="font-weight: bold">(</span>and subsequent dated model

releases<span style="font-weight: bold">)</span> or gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span> <span style="font-weight: bold">(</span>and

subsequent versions<span style="font-weight: bold">)</span>.

<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span> whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>

whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>

tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd

<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and dated model releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-

preview and dated model releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-

vision-preview, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k and dated model

releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo and dated model

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">9</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>

ENDP OINT

Models - OpenAI API

L ATE ST MODEL S

releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k and dated model

releases, fine-tuned versions of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span> <span style="font-weight: bold">(</span>Legacy<span style="font-weight: bold">)</span> gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct, babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>,

davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>

text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small, text-embedding-

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-large, text-embedding-ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo, babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>, davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>

<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>

text-moderation-stable, text-

<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo**

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is a sophisticated multimodal model capable of processing both text and image inputs to produce text outputs.
It is designed to tackle complex problems with higher accuracy than previous models, leveraging its extensive 
general knowledge and advanced reasoning skills. GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is accessible through the OpenAI API for paying customers 
and is optimized for chat applications, although it can also handle traditional completion tasks using the Chat 
Completions API.

**Model Versions:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview**
   - **Description:** This is the latest GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo model, designed to minimize instances where the model fails to
complete a task, known as <span style="color: #008000; text-decoration-color: #008000">"laziness."</span> It can return up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data:** Up to December <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview**
   - **Description:** This version currently points to the gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview model.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data:** Up to December <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-preview**
   - **Description:** This version of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo includes enhancements such as improved instruction following, 
JSON mode, reproducible outputs, and parallel function calling. It also supports up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data:** Up to April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

These models are part of OpenAI's ongoing efforts to provide developers with robust tools for various applications,
ensuring flexibility and improved performance across different use cases.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Models - OpenAI API Overview**

This document provides an overview of various GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> models, highlighting their capabilities, context windows, and 
training data timelines.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview**
   - **Description**: This model has the ability to understand images, in addition to all other GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo 
capabilities. It currently points to the gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-vision-preview model.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data**: Up to April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-vision-preview**
   - **Description**: Similar to the gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview, this model can understand images and includes all GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> 
Turbo capabilities. It returns a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens and is a preview model version.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data**: Up to April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>**
   - **Description**: This model currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> and includes continuous model upgrades.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span> tokens
   - **Training Data**: Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description**: A snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> from June 13th, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, with improved function calling support.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span> tokens
   - **Training Data**: Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k**
   - **Description**: This model points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> and includes continuous model upgrades. It was not widely
rolled out in favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens
   - **Training Data**: Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description**: A snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k from June 13th, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, with improved function calling support. Like 
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k, it was not widely rolled out in favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens
   - **Training Data**: Up to September 
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Multilingual Capabilities and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo**

**Multilingual Capabilities**

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> surpasses previous large language models and, as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, most state-of-the-art systems. It excels in the 
MMLU benchmark, which involves English-language multiple-choice questions across <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">57</span> subjects. GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> not only 
outperforms existing models in English but also shows strong performance in other languages.

**GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo**

GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo models are designed to understand and generate natural language or code. They are optimized for chat 
using the Chat Completions API but are also effective for non-chat tasks.

**Model Descriptions:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>**
   - **Description:** Updated GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo with improved accuracy and a fix for a text encoding bug in non-English
language function calls. It returns up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo**
   - **Description:** Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. The alias will automatically upgrade to 
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span> on February 16th.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>**
   - **Description:** Features improved instruction following, JSON mode, reproducible outputs, and parallel 
function calling. It returns up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Models - OpenAI API**

**GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Models:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct**
   - **Description:** Similar capabilities to GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> era models. Compatible with legacy Completions endpoint, not 
Chat Completions.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k**
   - **Description:** Legacy model pointing to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description:** Legacy snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo from June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Will be deprecated on June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description:** Legacy snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-turbo from June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Will be deprecated on June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>,
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>

**DALL-E:**

- DALL-E is an AI system that creates realistic images and art from natural language descriptions. DALL-E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> 
supports creating new images with specific sizes and editing existing images or creating variations. Available 
through the Images API and ChatGPT Plus.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>**
   - **Description:** The latest DALL-E model released in November <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>**
   - **Description:** Released in November <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2022</span>, this model offers more realistic, accurate, and higher resolution 
images than the original.

**TTS <span style="font-weight: bold">(</span>Text-to-Speech<span style="font-weight: bold">)</span>:**

- TTS converts text to natural-sounding spoken text. Two model variants are offered:
  - **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>:** Optimized for real-time text-to-speech use cases.
  - **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd:** Optimized for quality.
- These models can be used with the Speech endpoint in
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Models - OpenAI API**

**Text-to-Speech Models:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>**: This is a new text-to-speech model optimized for speed, providing efficient conversion of text into 
spoken words.
   
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd**: This model is optimized for quality, offering high-definition text-to-speech conversion.

**Whisper:**

Whisper is a versatile speech recognition model capable of handling diverse audio inputs. It supports multilingual 
speech recognition, speech translation, and language identification. The Whisper v2-large model is accessible via 
the API under the name <span style="color: #008000; text-decoration-color: #008000">"whisper-1."</span> While the open-source version and the API version are similar, the API offers 
an optimized inference process for faster performance. More technical details can be found in the associated paper.

**Embeddings:**

Embeddings are numerical representations of text, useful for measuring the relatedness between text pieces. They 
are applied in search, clustering, recommendations, anomaly detection, and classification tasks.

- **text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-large**: The most capable embedding model for both English and non-English tasks, with an 
output dimension of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">072</span>.
  
- **text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small**: Offers improved performance over the second-generation ada embedding model, with an 
output dimension of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>.
  
- **text-embedding-ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>**: A second-generation embedding model replacing <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span> first-generation models, also with 
an output dimension of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>.

**Moderation:**

The document mentions a section on moderation, likely related to content moderation capabilities, though specific 
details are not provided in the visible content.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Moderation Models and GPT Base**

**Moderation Models**

The moderation models are designed to ensure content compliance with OpenAI's usage policies. They classify content
into categories such as hate, hate/threatening, self-harm, sexual, sexual/minors, violence, and violence/graphic. 
These models process inputs by breaking them into chunks of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens. If the input exceeds <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens, some 
tokens may be truncated, potentially omitting a few from the moderation check.

The moderation endpoint provides the maximum score per category from each request. For instance, if one chunk 
scores <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span> and another scores <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1901</span> in a category, the API response will show <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span>.

- **text-moderation-latest**: Points to text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span> with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens.
- **text-moderation-stable**: Also points to text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span> with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens.
- **text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span>**: The most capable model across all categories with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens.

**GPT Base**

GPT base models are capable of understanding and generating natural language or code but are not trained for 
instruction following. They serve as replacements for the original GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> base models and utilize the legacy 
Completions API. Most users are advised to use GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> or GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>.

- **babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>**: Replaces the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> ada and babbage models, with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span> tokens and training data up to 
September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>.
- **davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>**: Replaces the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> curie and davinci models, with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span> tokens and training data up to
September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Your Data is Your Data

As of March <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, data sent to the OpenAI API is not used to train or improve OpenAI models unless you 
explicitly opt in. Opting in can help models improve for your specific use case over time.

To prevent abuse, API data may be retained for up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days before deletion, unless legally required otherwise. 
Trusted customers with sensitive applications may have zero data retention, meaning request and response bodies are
not logged and exist only in memory to serve the request.

This data policy does not apply to OpenAI's non-API consumer services like ChatGPT or DALL-E Labs.

**Default Usage Policies by Endpoint**

- **<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>**: Data is not used for training. Default retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, and it is eligible for 
zero retention except for image inputs.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">files</span>**: Data is not used for training. Retention is until deleted by the customer, with no zero retention 
option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>**: Data is not used for training. Retention is until deleted by the customer, with no zero 
retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">threads</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">messages</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">runs</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/threads/runs/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">steps</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">generations</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">edits</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">variations</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, and it is eligible for zero retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span>**: Data is not used for training
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">### Model Endpoint Compatibility and Data Retention

#### Data Retention Details

The table outlines the data retention policies for various API endpoints:

- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>**: No data is used for training, and there is zero data retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>**: No data is used for training, with a default retention period of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days. It is not 
eligible for zero retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>**: No data is used for training, and data is retained until deleted by the customer. It is
not eligible for zero retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>**: No data is used for training, and there is zero data retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>**: No data is used for training, with a default retention period of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days. It is eligible for
zero retention.

Additional notes:
- Image inputs via the `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview` model are not eligible for zero retention.
- The default retention period for the Assistants API is still being evaluated during the Beta phase.

#### Model Endpoint Compatibility

The table provides information on the compatibility of endpoints with the latest models:

- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>**: Supports all models except `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0301</span>`. The `retrieval` tool requires 
`gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview` or `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span>**: Compatible with `whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>**: Compatible with `whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>**: Compatible with `tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>` and `tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>**: Compatible with `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>`, `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview`, `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview`, `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k`, 
and `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo`.

For more details, users are encouraged to refer to the API data usage policies or contact the sales team for 
information on zero retention.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">LATEST MODELS

This document outlines the latest models available for different endpoints in the OpenAI API:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span> <span style="font-weight: bold">(</span>Legacy<span style="font-weight: bold">)</span>**:
   - Models: `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct`, `babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`, `davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`
   - These models are used for generating text completions based on input prompts.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>**:
   - Models: `text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small`, `text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-large`, `text-embedding-ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`
   - These models are designed to convert text into numerical vectors, which can be used for various tasks like 
similarity comparison and clustering.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>**:
   - Models: `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo`, `babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`, `davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`
   - These models support fine-tuning, allowing users to customize the models for specific tasks by training them 
on additional data.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>**:
   - Models: `text-moderation-stable`
   - This model is used for content moderation, helping to identify and filter out inappropriate or harmful 
content.

Additionally, the document mentions the availability of `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k` and other fine-tuned versions of 
`gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo`, indicating enhancements in model capabilities and performance.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Overview

Evaluation is the process of validating 
and testing the outputs that your LLM 
applications are producing. Having 
strong evaluations <span style="font-weight: bold">(</span>“evals”<span style="font-weight: bold">)</span> will mean a 
more stable, reliable application which is 
resilient to code and model changes.

Example use cases

- Quantify a solution’s reliability
- Monitor application performance in 

production
Test for regressions 

-

What we’ll cover

● What are evals

● Technical patterns

● Example framework

● Best practices

● Resources

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What are evals
Example

An evaluation contains a question and a correct answer. We call this the ground truth.

Question

What is the population 
of Canada?

Thought: I don’t know. I 
should use a tool
Action: Search
Action Input: What is the 
population of Canada?

LLM

Search

There are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people 
in Canada as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.

The current population of 
Canada is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> as of 
Tuesday, May <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>….

Actual result

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>




An evaluation, or <span style="color: #008000; text-decoration-color: #008000">"eval,"</span> involves a question and a correct answer, known as the ground truth. In this example, the
question posed is, <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span> 

The process begins with a person asking this question. The language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> initially does not know the answer 
and decides to use a tool to find it. The LLM takes the action of searching, with the input being the question 
about Canada's population.

The search tool then provides the answer: <span style="color: #008000; text-decoration-color: #008000">"The current population of Canada is 39,566,248 as of Tuesday, May 23, </span>
<span style="color: #008000; text-decoration-color: #008000">2023."</span> This result matches the actual result expected, which is that there are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people in Canada as of 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. 

This example illustrates how evaluations are used to verify the accuracy of information provided by a language 
model.

This slide provides an example of an evaluation process, often referred to as <span style="color: #008000; text-decoration-color: #008000">"evals."</span> The purpose of evals is to 
compare a predicted answer to a known correct answer, called the <span style="color: #008000; text-decoration-color: #008000">"ground truth,"</span> to determine if they match.

In this example, the question posed is: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span> The ground truth states that the 
population of Canada in <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people. The predicted answer is: <span style="color: #008000; text-decoration-color: #008000">"There are 39,566,248 people in Canada </span>
<span style="color: #008000; text-decoration-color: #008000">as of 2023."</span>

Since the predicted answer matches the ground truth, the evaluation is successful, as indicated by a checkmark. 
This process is crucial for verifying the accuracy of predictions in various applications.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What are evals
Example

Our ground truth matches the predicted answer, so the evaluation passes!

Evaluation

Question

Ground Truth

Predicted Answer

What is the population 
of Canada?

The population of Canada in 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people.

There are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people 
in Canada as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>




An evaluation, or <span style="color: #008000; text-decoration-color: #008000">"eval,"</span> involves a question and a correct answer, known as the ground truth. In this example, the
question posed is, <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span> 

The process begins with a person asking this question. The language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> initially does not know the answer 
and decides to use a tool to find it. The LLM takes the action of searching, with the input being the question 
about Canada's population.

The search tool then provides the answer: <span style="color: #008000; text-decoration-color: #008000">"The current population of Canada is 39,566,248 as of Tuesday, May 23, </span>
<span style="color: #008000; text-decoration-color: #008000">2023."</span> This result matches the actual result expected, which is that there are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people in Canada as of 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. 

This example illustrates how evaluations are used to verify the accuracy of information provided by a language 
model.

This slide provides an example of an evaluation process, often referred to as <span style="color: #008000; text-decoration-color: #008000">"evals."</span> The purpose of evals is to 
compare a predicted answer to a known correct answer, called the <span style="color: #008000; text-decoration-color: #008000">"ground truth,"</span> to determine if they match.

In this example, the question posed is: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span> The ground truth states that the 
population of Canada in <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people. The predicted answer is: <span style="color: #008000; text-decoration-color: #008000">"There are 39,566,248 people in Canada </span>
<span style="color: #008000; text-decoration-color: #008000">as of 2023."</span>

Since the predicted answer matches the ground truth, the evaluation is successful, as indicated by a checkmark. 
This process is crucial for verifying the accuracy of predictions in various applications.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns

Metric-based evaluations

Component evaluations

Subjective evaluations

●

●

Comparison metrics like 
BLEU, ROUGE

Gives a score to ﬁlter and 
rank results

●

●

Compares ground 
truth to prediction

Gives Pass/Fail

●

●

Uses a scorecard to 
evaluate subjectively

Scorecard may also 
have a Pass/Fail

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations

ROUGE is a common metric for evaluating machine summarizations of text

ROUGE

Metric for evaluating 
summarization tasks

Original

OpenAI's mission is to ensure that 
artiﬁcial general intelligence <span style="font-weight: bold">(</span>AGI<span style="font-weight: bold">)</span> 
beneﬁts all of humanity. OpenAI 
will build safe and beneﬁcial AGI 
directly, but will also consider its 
mission fulﬁlled if its work aids 
others to achieve this outcome. 
OpenAI follows several key 
principles for this purpose. First, 
broadly distributed beneﬁts - any 
inﬂuence over AGI's deployment 
will be used for the beneﬁt of all, 
and to avoid harmful uses or undue 
concentration of power…

Machine 
Summary

OpenAI aims to ensure AGI is 
for everyone's use, totally 
avoiding harmful stuff or big 
power concentration. 
Committed to researching 
AGI's safe side, promoting 
these studies in AI folks. 
OpenAI wants to be top in AI 
things and works with 
worldwide research, policy 
groups to ﬁgure AGI's stuff.

ROUGE 
Score

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.51162</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">7</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations

BLEU score is another standard metric, this time focusing on machine translation tasks

BLEU

Original text

Reference
Translation

Predicted 
Translation

Metric for 
evaluating 
translation tasks

Y gwir oedd 
doedden nhw 
ddim yn dweud 
celwyddau wedi'r 
cwbl.

The truth was 
they were not 
telling lies after 
all.

The truth was 
they weren't 
telling lies after 
all.

BLEU 
Score

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.39938</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations

What they’re good for

What to be aware of

●

●

A good starting point for evaluating a 

● Not tuned to your speciﬁc context

fresh solution

Useful yardstick for automated testing 

of whether a change has triggered a 

major performance shift

● Most customers require more 

sophisticated evaluations to go to 

production

● Cheap and fast

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">9</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Component evaluations

Component evaluations <span style="font-weight: bold">(</span>or “unit tests”<span style="font-weight: bold">)</span> cover a single input/output of the application. They check 
whether each component works in isolation, comparing the input to a ground truth ideal result

Is this the 
correct action?

Exact match 
comparison

Does this answer 
use the context?

Extract numbers 
from each and 
compare

What is the population 
of Canada?

Thought: I don’t know. I 
should use a tool
Action: Search
Action Input: What is the 
population of Canada?

Agent

Search

There are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people 
in Canada as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.

The current population of 
Canada is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> as of 
Tuesday, May <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>….

Is this the right 
search result?

Tag the right 
answer and do 
an exact match 
comparison with 
the retrieval.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Subjective evaluations

Building up a good scorecard for automated testing beneﬁts from a few rounds of detailed human 
review so we can learn what is valuable. 

A policy of “show rather than tell” is also advised for GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, so include examples of what a <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> and 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> out of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> look like so the model can appreciate the spread.

Example 
scorecard

You are a helpful evaluation assistant who grades how well the Assistant has answered the customer’s query.

You will assess each submission against these metrics, please think through these step by step:

-

relevance: Grade how relevant the search content is to the question from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> <span style="color: #800080; text-decoration-color: #800080">//</span> <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> being highly relevant and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> 
being 
not relevant at all.

- credibility: Grade how credible the sources provided are from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> <span style="color: #800080; text-decoration-color: #800080">//</span> <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> being an established newspaper, 

-

government agency or large company and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> being unreferenced.
result: Assess whether the question is correct given only the content returned from the search and the user’s 
question <span style="color: #800080; text-decoration-color: #800080">//</span> acceptable values are “correct” or “incorrect”

You will output this as a JSON document: <span style="font-weight: bold">{</span>relevance: integer, credibility: integer, result: string<span style="font-weight: bold">}</span>

User: What is the population of Canada?
Assistant: Canada's population was estimated at <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">858</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">480</span> on April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> by Statistics Canada.
Evaluation: <span style="font-weight: bold">{</span>relevance: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, credibility: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, result: correct<span style="font-weight: bold">}</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">11</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Example framework

Your evaluations can be grouped up into test suites called runs and executed in a batch to test 
the eﬀectiveness of your system.

Each run should have its contents logged and stored at the most granular level possible 
<span style="font-weight: bold">(</span>“tracing”<span style="font-weight: bold">)</span> so you can investigate failure reasons, make tweaks and then rerun your evals.

Run ID Model

Score

Annotation feedback

Changes since last run

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">28</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">36</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">34</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>

● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">18</span> incorrect with correct search results
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches

N/A

● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> incorrect with correct search results
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches

● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">12</span> incorrect with correct search results
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches

Model updated to GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

Added few-shot examples

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">42</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>

● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> incorrect with correct search results

Added metadata to search
Prompt engineering for Answer step

gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">48</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>

● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> incorrect with correct search results

Prompt engineering to Answer step

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">12</span>




This diagram illustrates a framework for processing a return request using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> system. Here's a 
breakdown of the process:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **User Input**: The user wants to return a T-shirt purchased on Amazon on March 3rd.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Router**: The initial input is processed by a router LLM, which determines the nature of the request. The 
expected and predicted outcomes are both <span style="color: #008000; text-decoration-color: #008000">"return,"</span> and the process passes this evaluation.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Return Assistant**: The request is then handled by a return assistant LLM. It interacts with a knowledge base 
to verify the return policy.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Knowledge Base**: The system checks the return policy, confirming that the item is eligible for return within 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days of purchase. The expected and predicted outcomes are <span style="color: #008000; text-decoration-color: #008000">"return_policy,"</span> and this step also passes.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>. **Response to User**: The system responds to the user, confirming that the return can be processed because it is
within the <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>-day window.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>. **Evaluation**: The response is evaluated for adherence to guidelines, scoring <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> for politeness, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for 
coherence, and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for relevancy, resulting in a pass.

The framework uses both component evaluations <span style="font-weight: bold">(</span>red dashed lines<span style="font-weight: bold">)</span> and subjective evaluations <span style="font-weight: bold">(</span>orange dashed lines<span style="font-weight: bold">)</span> 
to ensure the process is accurate and user-friendly.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Example framework

I want to return a 
T-shirt I bought on 
Amazon on March 3rd.

User

Router

LLM

Expected: return
Predicted: return
PASS

Return
Assistant

LLM

Component evals

Subjective evals

Expected: return_policy
Predicted: return_policy
PASS

Knowledge 
base

Question: Does this response adhere to 
our guidelines
Score: 
Politeness: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, Coherence: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, Relevancy: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
PASS

Sure - because we’re 
within <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days of the 
purchase, I can 
process the return

Question: I want to return a T-shirt I 
bought on Amazon on March 3rd.
Ground truth: Eligible for return
PASS

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>




This diagram illustrates a framework for processing a return request using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> system. Here's a 
breakdown of the process:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **User Input**: The user wants to return a T-shirt purchased on Amazon on March 3rd.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Router**: The initial input is processed by a router LLM, which determines the nature of the request. The 
expected and predicted outcomes are both <span style="color: #008000; text-decoration-color: #008000">"return,"</span> and the process passes this evaluation.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Return Assistant**: The request is then handled by a return assistant LLM. It interacts with a knowledge base 
to verify the return policy.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Knowledge Base**: The system checks the return policy, confirming that the item is eligible for return within 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days of purchase. The expected and predicted outcomes are <span style="color: #008000; text-decoration-color: #008000">"return_policy,"</span> and this step also passes.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>. **Response to User**: The system responds to the user, confirming that the return can be processed because it is
within the <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>-day window.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>. **Evaluation**: The response is evaluated for adherence to guidelines, scoring <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> for politeness, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for 
coherence, and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for relevancy, resulting in a pass.

The framework uses both component evaluations <span style="font-weight: bold">(</span>red dashed lines<span style="font-weight: bold">)</span> and subjective evaluations <span style="font-weight: bold">(</span>orange dashed lines<span style="font-weight: bold">)</span> 
to ensure the process is accurate and user-friendly.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Best practices

Log everything

●

Evals need test cases - log everything as you develop so you can mine your logs for good eval cases

Create a feedback loop

●
●

Build evals into your application so you can quickly run them, iterate and rerun to see the impact
Evals also provide a useful structure for few-shot or ﬁne-tuning examples when optimizing

Employ expert labellers who know the process

● Use experts to help create your eval cases - these need to be as lifelike as possible

Evaluate early and often

●

Evals are something you should build as soon as you have your ﬁrst functioning prompt - you won’t be 
able to optimize without this baseline, so build it early

● Making evals early also forces you to engage with what a good response looks like




<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Log Everything**
   - It's important to log all test cases during development. This allows you to mine your logs for effective 
evaluation cases.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Create a Feedback Loop**
   - Integrate evaluations into your application to quickly run, iterate, and rerun them to observe impacts.
   - Evaluations provide a useful structure for few-shot or fine-tuning examples during optimization.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Employ Expert Labelers Who Know the Process**
   - Use experts to help create evaluation cases, ensuring they are as realistic as possible.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Evaluate Early and Often**
   - Build evaluations as soon as you have a functioning prompt. This baseline is crucial for optimization.
   - Early evaluations help you understand what a good response looks like, facilitating better engagement.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">## Overview

Evaluation is the process of validating and testing the outputs that your Large Language Model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> applications 
are producing. Strong evaluations, referred to as <span style="color: #008000; text-decoration-color: #008000">"evals,"</span> contribute to creating a more stable and reliable 
application that can withstand changes in code and model updates.

### Example Use Cases
- **Quantify a solution’s reliability**: Measure how dependable your application is.
- **Monitor application performance in production**: Keep track of how well your application performs in real-world
scenarios.
- **Test for regressions**: Ensure that new updates do not negatively impact existing functionality.

### What We’ll Cover
- **What are evals**: Understanding the concept and importance of evaluations.
- **Technical patterns**: Exploring common methods and strategies used in evaluations.
- **Example framework**: Providing a structured approach to implementing evaluations.
- **Best practices**: Sharing tips and guidelines for effective evaluations.
- **Resources**: Offering additional materials for further learning and exploration.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns**

This slide outlines three types of evaluation methods used in technical assessments:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Metric-based Evaluations**:
   - These evaluations use comparison metrics such as BLEU and ROUGE. 
   - They provide a score that helps in filtering and ranking results, making it easier to assess the quality of 
outputs quantitatively.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Component Evaluations**:
   - This method involves comparing the ground truth to predictions.
   - It results in a simple Pass/Fail outcome, which is useful for determining whether specific components meet the
required standards.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Subjective Evaluations**:
   - These evaluations rely on a scorecard to assess outputs subjectively.
   - The scorecard can also include a Pass/Fail option, allowing for a more nuanced evaluation that considers 
qualitative aspects.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Metric-based Evaluations

ROUGE is a common metric for evaluating machine summarizations of text. It is specifically used to assess the 
quality of summaries by comparing them to reference summaries. The slide provides an example of how ROUGE is 
applied:

- **Original Text**: This is a detailed description of OpenAI's mission, emphasizing the development of artificial 
general intelligence <span style="font-weight: bold">(</span>AGI<span style="font-weight: bold">)</span> that benefits humanity. It highlights the importance of safety, broad distribution of 
benefits, and avoiding harmful uses or power concentration.

- **Machine Summary**: This is a condensed version of the original text. It focuses on ensuring AGI is safe and 
accessible, avoiding harm and power concentration, and promoting research and collaboration in AI.

- **ROUGE Score**: The score given is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.51162</span>, which quantifies the similarity between the machine-generated 
summary and the original text. A higher score indicates a closer match to the reference summary.

Overall, ROUGE helps in evaluating how well a machine-generated summary captures the essence of the original text.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"># Technical Patterns: Metric-based Evaluations

The slide discusses the BLEU score, a standard metric used to evaluate machine translation tasks. BLEU stands for 
Bilingual Evaluation Understudy and is a method for assessing the quality of text that has been machine-translated 
from one language to another.

### Key Elements:

- **BLEU**: This is a metric specifically designed for evaluating translation tasks. It compares the 
machine-generated translation to one or more reference translations.

- **Original Text**: The example given is in Welsh: <span style="color: #008000; text-decoration-color: #008000">"Y gwir oedd doedden nhw ddim yn dweud celwyddau wedi'r cwbl."</span>

- **Reference Translation**: This is the human-generated translation used as a standard for comparison: <span style="color: #008000; text-decoration-color: #008000">"The truth </span>
<span style="color: #008000; text-decoration-color: #008000">was they were not telling lies after all."</span>

- **Predicted Translation**: This is the translation produced by the machine: <span style="color: #008000; text-decoration-color: #008000">"The truth was they weren't telling </span>
<span style="color: #008000; text-decoration-color: #008000">lies after all."</span>

- **BLEU Score**: The score for this translation is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.39938</span>. This score indicates how closely the machine 
translation matches the reference translation, with a higher score representing a closer match.

The BLEU score is widely used in the field of natural language processing to provide a quantitative measure of 
translation quality.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Metric-based Evaluations

**What they’re good for:**

- **Starting Point**: They provide a good starting point for evaluating a new solution, helping to establish 
initial benchmarks.
- **Automated Testing**: These evaluations serve as a useful yardstick for automated testing, particularly in 
determining if a change has caused a significant performance shift.
- **Cost-Effective**: They are cheap and fast, making them accessible for quick assessments.

**What to be aware of:**

- **Context Specificity**: These evaluations are not tailored to specific contexts, which can limit their 
effectiveness in certain situations.
- **Sophistication Needs**: Most customers require more sophisticated evaluations before moving to production, 
indicating that metric-based evaluations might not be sufficient on their own for final decision-making.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Component Evaluations**

Component evaluations, also known as <span style="color: #008000; text-decoration-color: #008000">"unit tests,"</span> focus on assessing a single input/output of an application. The 
goal is to verify that each component functions correctly in isolation by comparing the input to a predefined ideal
result, known as the ground truth.

**Process Overview:**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Input Question:** 
   - The process begins with a question: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Agent's Role:**
   - The agent receives the question and processes it. The agent's thought process is: <span style="color: #008000; text-decoration-color: #008000">"I don’t know. I should use </span>
<span style="color: #008000; text-decoration-color: #008000">a tool."</span>
   - The agent decides on an action: <span style="color: #008000; text-decoration-color: #008000">"Search."</span>
   - The action input is the original question: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Search Component:**
   - The search component is tasked with finding the answer. It retrieves the information: <span style="color: #008000; text-decoration-color: #008000">"The current population </span>
<span style="color: #008000; text-decoration-color: #008000">of Canada is 39,566,248 as of Tuesday, May 23, 2023…."</span>

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Evaluation Steps:**
   - **Correct Action Check:** Is the agent's decision to search the correct action?
   - **Exact Match Comparison:** Does the retrieved answer match the expected result exactly?
   - **Contextual Relevance:** Does the answer use the context provided in the question?
   - **Number Extraction and Comparison:** Extract numbers from both the expected and retrieved answers and compare
them for accuracy.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>. **Final Output:**
   - The final output is the verified answer: <span style="color: #008000; text-decoration-color: #008000">"There are 39,566,248 people in Canada as of 2023."</span>

This process ensures that each component of the application is functioning correctly and producing accurate results
by systematically evaluating each step against the ground truth.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Subjective Evaluations**

Building an effective scorecard for automated testing is enhanced by incorporating detailed human reviews. This 
process helps identify what is truly valuable. The approach of <span style="color: #008000; text-decoration-color: #008000">"show rather than tell"</span> is recommended for GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, 
meaning that examples of scores like <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>, and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> out of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> should be provided to help the model understand the 
range.

**Example Scorecard:**

- **Role**: You are an evaluation assistant assessing how well the Assistant has answered a customer's query.
  
- **Metrics for Assessment**:
  - **Relevance**: Rate the relevance of the search content to the question on a scale from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, where <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> is 
highly relevant and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> is not relevant at all.
  - **Credibility**: Rate the credibility of the sources from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, where <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> is an established newspaper, 
government agency, or large company, and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> is unreferenced.
  - **Result**: Determine if the question is answered correctly based on the search content and the user's 
question. Acceptable values are <span style="color: #008000; text-decoration-color: #008000">"correct"</span> or <span style="color: #008000; text-decoration-color: #008000">"incorrect."</span>

- **Output Format**: Provide the evaluation as a JSON document with fields for relevance, credibility, and result.

**Example Evaluation**:
- **User Query**: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>
- **Assistant's Response**: <span style="color: #008000; text-decoration-color: #008000">"Canada's population was estimated at 39,858,480 on April 1, 2023, by Statistics </span>
<span style="color: #008000; text-decoration-color: #008000">Canada."</span>
- **Evaluation**: `<span style="font-weight: bold">{</span>relevance: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, credibility: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, result: correct<span style="font-weight: bold">}</span>`

This structured approach ensures clarity and consistency in evaluating the performance of automated systems.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Example Framework**

This framework outlines a method for evaluating the effectiveness of a system by grouping evaluations into test 
suites called <span style="color: #008000; text-decoration-color: #008000">"runs."</span> These runs are executed in batches, and each run's contents are logged and stored at a 
detailed level, known as <span style="color: #008000; text-decoration-color: #008000">"tracing."</span> This allows for investigation of failures, making adjustments, and rerunning 
evaluations.

The table provides a summary of different runs:

- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>**: 
  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">28</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">18</span> incorrect with correct search results, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
  - Changes: N/A

- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>**: 
  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">36</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> incorrect with correct search results, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
  - Changes: Model updated to GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>

- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>**: 
  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">34</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">12</span> incorrect with correct search results, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
  - Changes: Added few-shot examples

- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>**: 
  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">42</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> incorrect with correct search results
  - Changes: Added metadata to search, Prompt engineering for Answer step

- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>**: 
  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">48</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> incorrect with correct search results
  - Changes: Prompt engineering to Answer step

This framework emphasizes the importance of detailed logging and iterative improvements to enhance system 
performance.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Overview

Fine-tuning involves adjusting the 
parameters of pre-trained models on a 
speciﬁc dataset or task. This process 
enhances the model's ability to generate 
more accurate and relevant responses for 
the given context by adapting it to the 
nuances and speciﬁc requirements of the 
task at hand.

Example use cases

- Generate output in a consistent 

-

format
Process input by following speciﬁc 
instructions

What we’ll cover

● When to ﬁne-tune

● Preparing the dataset

● Best practices

● Hyperparameters

● Fine-tuning advances

● Resources

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What is Fine-tuning

Public Model

Training data

Training

Fine-tuned 
model

Fine-tuning a model consists of training the 
model to follow a set of given input/output 
examples.

This will teach the model to behave in a 
certain way when confronted with a similar 
input in the future.

We recommend using <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples 

even if the minimum is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>




Fine-tuning is a process in machine learning where a pre-existing model, known as a public model, is further 
trained using specific training data. This involves adjusting the model to follow a set of given input/output 
examples. The goal is to teach the model to respond in a particular way when it encounters similar inputs in the 
future.

The diagram illustrates this process: starting with a public model, training data is used in a training phase to 
produce a fine-tuned model. This refined model is better suited to specific tasks or datasets.

It is recommended to use <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples for effective fine-tuning, although the minimum requirement is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> 
examples. This ensures the model learns adequately from the examples provided.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">When to ﬁne-tune

Good for  ✅

Not good for  ❌

●

●

●

●

Following a given format or tone for the 

output

Processing the input following speciﬁc, 

complex instructions

Improving latency

Reducing token usage

●

●

●

Teaching the model new knowledge
➔ Use RAG or custom models instead

Performing well at multiple, unrelated tasks
➔ Do prompt-engineering or create multiple 

FT models instead

Include up-to-date content in responses
➔ Use RAG instead

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Preparing the dataset

Example format

<span style="font-weight: bold">{</span>

<span style="color: #008000; text-decoration-color: #008000">"messages"</span>: <span style="font-weight: bold">[</span>

<span style="font-weight: bold">{</span>

<span style="color: #008000; text-decoration-color: #008000">"role"</span>: <span style="color: #008000; text-decoration-color: #008000">"system"</span>,
<span style="color: #008000; text-decoration-color: #008000">"content"</span>: "Marv is a factual chatbot 
that is also sarcastic."

<span style="font-weight: bold">}</span>,
<span style="font-weight: bold">{</span>

<span style="color: #008000; text-decoration-color: #008000">"role"</span>: <span style="color: #008000; text-decoration-color: #008000">"user"</span>,
<span style="color: #008000; text-decoration-color: #008000">"content"</span>: "What's the capital of 
France?"

<span style="font-weight: bold">}</span>,
<span style="font-weight: bold">{</span>

<span style="color: #008000; text-decoration-color: #008000">"role"</span>: <span style="color: #008000; text-decoration-color: #008000">"assistant"</span>,
<span style="color: #008000; text-decoration-color: #008000">"content"</span>: "Paris, as if everyone 
doesn't know that already."

<span style="font-weight: bold">}</span>

<span style="font-weight: bold">]</span>

<span style="font-weight: bold">}</span>

.jsonl

➔ Take the set of instructions and prompts that you 

found worked best for the model prior to ﬁne-tuning. 
Include them in every training example

➔ If you would like to shorten the instructions or 

prompts, it may take more training examples to arrive 
at good results

We recommend using <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples 

even if the minimum is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">6</span>



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Best practices

Curate examples carefully

Datasets can be diﬃcult to build, start 
small and invest intentionally. 
Optimize for fewer high-quality 
training examples.

● Consider “prompt baking”, or using a basic 
prompt to generate your initial examples
● If your conversations are multi-turn, ensure 

your examples are representative

● Collect examples to target issues detected 

in evaluation

● Consider the balance &#x26; diversity of data
● Make sure your examples contain all the 

information needed in the response

Iterate on hyperparameters

Establish a baseline

Start with the defaults and adjust 
based on performance.

● If the model does not appear to converge, 

increase the learning rate multiplier
● If the model does not follow the training 
data as much as expected increase the 
number of epochs

● If the model becomes less diverse than 

expected decrease the # of epochs by <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>

Automate your feedback 
pipeline

Introduce automated evaluations to 
highlight potential problem cases to 
clean up and use as training data.

Consider the G-Eval approach of 
using GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> to perform automated 
testing using a scorecard.

Often users start with a 
zero-shot or few-shot prompt to 
build a baseline evaluation 
before graduating to ﬁne-tuning.

Often users start with a 
zero-shot or few-shot prompt to 
build a baseline evaluation 
Optimize for latency and 
before graduating to ﬁne-tuning.
token eﬃciency

When using GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, once you 
have a baseline evaluation and 
training examples consider 
ﬁne-tuning <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> to get similar 
performance for less cost and 
latency.

Experiment with reducing or 
removing system instructions 
with subsequent ﬁne-tuned 
model versions.



</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Hyperparameters

Epochs
Refers to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> full cycle through the training dataset
If you have hundreds of thousands of examples, we would recommend 
experimenting with two epochs <span style="font-weight: bold">(</span>or one<span style="font-weight: bold">)</span> to avoid overﬁtting.

default: auto <span style="font-weight: bold">(</span>standard is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span><span style="font-weight: bold">)</span>

Batch size
Number of training examples used to train a single 
forward &#x26; backward pass
In general, we've found that larger batch sizes tend to work better for larger datasets

default: ~<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>% x N* <span style="font-weight: bold">(</span>max <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">256</span><span style="font-weight: bold">)</span>

*N = number of training examples

Learning rate multiplier
Scaling factor for the original learning rate
We recommend experimenting with values between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.02</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>. We've found that 
larger learning rates often perform better with larger batch sizes.

default: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.05</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1</span> or <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>*

*depends on ﬁnal batch size

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>




**Epochs**
- An epoch refers to one complete cycle through the training dataset.
- For datasets with hundreds of thousands of examples, it is recommended to use fewer epochs <span style="font-weight: bold">(</span>one or two<span style="font-weight: bold">)</span> to 
prevent overfitting.
- Default setting is auto, with a standard of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> epochs.

**Batch Size**
- This is the number of training examples used to train in a single forward and backward pass.
- Larger batch sizes are generally more effective for larger datasets.
- The default batch size is approximately <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>% of the total number of training examples <span style="font-weight: bold">(</span>N<span style="font-weight: bold">)</span>, with a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">256</span>.

**Learning Rate Multiplier**
- This is a scaling factor for the original learning rate.
- Experimentation with values between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.02</span> and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span> is recommended.
- Larger learning rates often yield better results with larger batch sizes.
- Default values are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.05</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1</span>, or <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>, depending on the final batch size.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Overview**

Fine-tuning involves adjusting the parameters of pre-trained models on a specific dataset or task. This process 
enhances the model's ability to generate more accurate and relevant responses for the given context by adapting it 
to the nuances and specific requirements of the task at hand.

**Example Use Cases:**
- Generate output in a consistent format.
- Process input by following specific instructions.

**What We’ll Cover:**
- When to fine-tune
- Preparing the dataset
- Best practices
- Hyperparameters
- Fine-tuning advances
- Resources
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">When to Fine-Tune

**Good for:**

- **Following a given format or tone for the output:** Fine-tuning is effective when you need the model to adhere 
to a specific style or structure in its responses.
  
- **Processing the input following specific, complex instructions:** It helps in handling detailed and intricate 
instructions accurately.

- **Improving latency:** Fine-tuning can enhance the speed of the model's responses.

- **Reducing token usage:** It can optimize the model to use fewer tokens, making it more efficient.

**Not good for:**

- **Teaching the model new knowledge:** Fine-tuning is not suitable for adding new information to the model. 
Instead, use Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span> or custom models.

- **Performing well at multiple, unrelated tasks:** For diverse tasks, it's better to use prompt engineering or 
create multiple fine-tuned models.

- **Including up-to-date content in responses:** Fine-tuning is not ideal for ensuring the model has the latest 
information. RAG is recommended for this purpose.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Preparing the Dataset**

This slide provides guidance on preparing a dataset for training a chatbot model. It includes an example format 
using JSONL <span style="font-weight: bold">(</span>JSON Lines<span style="font-weight: bold">)</span> to structure the data. The example shows a conversation with three roles:

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **System**: Sets the context by describing the chatbot as <span style="color: #008000; text-decoration-color: #008000">"Marv is a factual chatbot that is also sarcastic."</span>
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **User**: Asks a question, <span style="color: #008000; text-decoration-color: #008000">"What's the capital of France?"</span>
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Assistant**: Responds with a sarcastic answer, <span style="color: #008000; text-decoration-color: #008000">"Paris, as if everyone doesn't know that already."</span>

Key recommendations for dataset preparation include:

- Use a set of instructions and prompts that have proven effective for the model before fine-tuning. These should 
be included in every training example.
- If you choose to shorten instructions or prompts, be aware that more training examples may be needed to achieve 
good results.
- It is recommended to use <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples, even though the minimum required is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Best Practices**

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>. **Curate Examples Carefully**
   - Building datasets can be challenging, so start small and focus on high-quality examples.
   - Use <span style="color: #008000; text-decoration-color: #008000">"prompt baking"</span> to generate initial examples.
   - Ensure multi-turn conversations are well-represented.
   - Collect examples to address issues found during evaluation.
   - Balance and diversify your data.
   - Ensure examples contain all necessary information for responses.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **Iterate on Hyperparameters**
   - Begin with default settings and adjust based on performance.
   - Increase the learning rate multiplier if the model doesn't converge.
   - Increase the number of epochs if the model doesn't follow training data closely.
   - Decrease the number of epochs by <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> if the model becomes less diverse.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>. **Establish a Baseline**
   - Start with zero-shot or few-shot prompts to create a baseline before fine-tuning.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>. **Automate Your Feedback Pipeline**
   - Use automated evaluations to identify and clean up problem cases for training data.
   - Consider using the G-Eval approach with GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for automated testing with a scorecard.

<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>. **Optimize for Latency and Token Efficiency**
   - After establishing a baseline, consider fine-tuning with GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> for similar performance at lower cost and 
latency.
   - Experiment with reducing or removing system instructions in subsequent fine-tuned versions.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

```python
# Cleaning up content
# Removing trailing spaces, additional line breaks, page numbers and references to the content being a slide
clean_content = []
for c in content:
    text = c.replace(' \n', '').replace('\n\n', '\n').replace('\n\n\n', '\n').strip()
    text = re.sub(r"(?<=\n)\d{1,2}", "", text)
    text = re.sub(r"\b(?:the|this)\s*slide\s*\w+\b", "", text, flags=re.IGNORECASE)
    clean_content.append(text)
```

```python
for c in clean_content:
    print(c)
    print("\n\n-------------------------------\n\n")
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Overview
Retrieval-Augmented Generationenhances the capabilities of languagemodels by combining them with aretrieval system.
This allows the modelto leverage external knowledge sourcesto generate more accurate andcontextually relevant 
responses.
Example use cases
- Provide answers with up-to-date
information
- Generate contextual responses
What we’ll cover
● Technical patterns
● Best practices
● Common pitfalls
● Resources

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What is RAG
Retrieve information to Augment the model’s knowledge and Generate the output
“What is yourreturn policy?”
ask
result
search
LLM
return information
Total refunds: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days
% of value vouchers: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days
$<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> discount on next order: > <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days
“You can get a full refund upto <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days after thepurchase, then up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> daysyou would get a voucher forhalf the 
value of your order”
KnowledgeBase <span style="color: #800080; text-decoration-color: #800080">/</span> Externalsources

RAG stands for <span style="color: #008000; text-decoration-color: #008000">"Retrieve information to Augment the model’s knowledge and Generate the output."</span> This process 
involves using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> to enhance its responses by accessing external information sources.
Here's how it works:
. **User Query**: A user asks a question, such as <span style="color: #008000; text-decoration-color: #008000">"What is your return policy?"</span>
. **LLM Processing**: The language model receives the question and initiates a search for relevant information.
. **Information Retrieval**: The LLM accesses a knowledge base or external sources to find the necessary details. 
In this example, the information retrieved includes:
   - Total refunds available from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days.
   - <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>% value vouchers for returns between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days.
   - A $<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> discount on the next order for returns after <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days.
. **Response Generation**: The LLM uses the retrieved information to generate a coherent response for the user. For
instance, it might say, <span style="color: #008000; text-decoration-color: #008000">"You can get a full refund up to 14 days after the purchase, then up to 30 days you would </span>
<span style="color: #008000; text-decoration-color: #008000">get a voucher for half the value of your order."</span>
This method allows the model to provide accurate and up-to-date answers by leveraging external data sources.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">When to use RAG
Good for  ✅
Not good for  ❌
●
●
Introducing new information to the model
●
Teaching the model a speciﬁc format, style,
to update its knowledge
Reducing hallucinations by controlling
content
<span style="color: #800080; text-decoration-color: #800080">/</span>!\ Hallucinations can still happen with RAG
or language
➔ Use ﬁne-tuning or custom models instead
●
Reducing token usage
➔ Consider ﬁne-tuning depending on the use
case

**Good for:**
- **Introducing new information to the model:** RAG <span style="font-weight: bold">(</span>Retrieval-Augmented Generation<span style="font-weight: bold">)</span> is effective for updating a 
model's knowledge by incorporating new data.
- **Reducing hallucinations by controlling content:** While RAG can help minimize hallucinations, it's important to
note that they can still occur.
**Not good for:**
- **Teaching the model a specific format, style, or language:** For these tasks, it's better to use fine-tuning or 
custom models.
- **Reducing token usage:** If token usage is a concern, consider fine-tuning based on the specific use case.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation
Input processing
Retrieval
Answer Generation
● Chunking
●
●
Embeddings
Augmentingcontent
●
Inputaugmentation
● NER
●
Search
● Context window
● Multi-stepretrieval
● Optimisation
●
Safety checks
●
Embeddings
● Re-ranking

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation
chunk documents into multiplepieces for easier consumption
content
embeddings
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">876</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.145</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.179</span>…
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…
Augment contentusing LLMs
Ex: parse text only, ask gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> to rephrase &#x26;summarize each part, generate bullet points…
BEST PRACTICES
Pre-process content for LLMconsumption:Add summary, headers for eachpart, etc.
+ curate relevant data sources
KnowledgeBase
COMMON PITFALLS
➔ Having too much low-quality
content
➔ Having too large documents

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation: chunking
Why chunking?
If your system doesn’t requireentire documents to providerelevant answers, you canchunk them into multiple 
piecesfor easier consumption <span style="font-weight: bold">(</span>reducedcost &#x26; latency<span style="font-weight: bold">)</span>.
Other approaches: graphs ormap-reduce
Things to consider
●
Overlap:
○
○
Should chunks be independent or overlap oneanother?
If they overlap, by how much?
●
Size of chunks:
○ What is the optimal chunk size for my use case?
○
Do I want to include a lot in the context window orjust the minimum?
● Where to chunk:
○
○
Should I chunk every N tokens or use speciﬁcseparators?Is there a logical way to split the context that wouldhelp 
the retrieval process?
● What to return:
○
○
Should I return chunks across multiple documentsor top chunks within the same doc?
Should chunks be linked together with metadata toindicate common properties?

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation: embeddings
What to embed?
Depending on your use caseyou might not want just toembed the text in thedocuments but metadata as well- anything 
that will make it easierto surface this speciﬁc chunk ordocument when performing asearch
Examples
Embedding Q&#x26;A posts in a forum
You might want to embed the title of the posts,the text of the original question and the content ofthe top answers.
Additionally, if the posts are tagged by topic orwith keywords, you can embed those too.
Embedding product specs
In additional to embedding the text contained indocuments describing the products, you mightwant to add metadata 
that you have on theproduct such as the color, size, etc. in yourembeddings.

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Data preparation: augmenting content
What does “Augmentingcontent” mean?
Augmenting content refers tomodiﬁcations of the original contentto make it more digestible for asystem relying on 
RAG. Themodiﬁcations could be a change informat, wording, or addingdescriptive content such assummaries or 
keywords.
Example approaches
Make it a guide*
Reformat the content to look more likea step-by-step guide with clearheadings and bullet-points, as thisformat is 
more easily understandableby an LLM.
Add descriptive metadata*
Consider adding keywords or text thatusers might search for when thinkingof a speciﬁc product or service.
Multimodality
Leverage modelssuch as Whisper orGPT-4V totransform audio orvisual content intotext.
For example, youcan use GPT-4V togenerate tags forimages or todescribe slides.
* GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can do this for you with the right prompt

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Input processing
Process input according to task
Q&#x26;A
HyDE:  Ask LLM to hypothetically answer thequestion &#x26; use the answer to search the KB
embeddings
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">876</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.145</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.179</span>…
Content search
Prompt LLM to rephrase input &#x26; optionally addmore context
query
SELECT * from items…
DB search
NER:  Find relevant entities to be used for akeyword search or to construct a search query
keywords
red
summer
BEST PRACTICES
Consider how to transform theinput to match content in thedatabase
Consider using metadata toaugment the user input
COMMON PITFALLS
➔ Comparing directly the inputto the database withoutconsidering the taskspeciﬁcities

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Input processing: input augmentation
What is input augmentation?
Example approaches
Augmenting the input means turningit into something diﬀerent, eitherrephrasing it, splitting it in severalinputs or
expanding it.
This helps boost performance asthe LLM might understand betterthe user intent.
Queryexpansion*
Rephrase thequery to bemoredescriptive
HyDE*
Hypotheticallyanswer thequestion &#x26; usethe answer tosearch the KB
Splitting a query in N*
When there is more than <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> question orintent in a user query, considersplitting it in several queries
Fallback
Considerimplementing aﬂow where the LLMcan ask forclariﬁcation whenthere is not enoughinformation in theoriginal 
user queryto get a result
<span style="font-weight: bold">(</span>Especially relevantwith tool usage<span style="font-weight: bold">)</span>
* GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can do this for you with the right prompt

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Input processing: NER
Why use NER?
Using NER <span style="font-weight: bold">(</span>Named EntityRecognition<span style="font-weight: bold">)</span> allows to extractrelevant entities from the input, thatcan then be used for 
moredeterministic search queries.This can be useful when the scopeis very constrained.
Example
Searching for movies
If you have a structured database containingmetadata on movies, you can extract genre,actors or directors names, 
etc. from the userquery and use this to search the database
Note: You can use exact values or embeddings afterhaving extracted the relevant entities

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval
re-ranking
INPUT
embeddings
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span>…
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">876</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.145</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.179</span>…
query
SELECT * from items…
keywords
red
summer
Semanticsearch
RESULTS
RESULTS
vector DB
relational <span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">nosql</span> db
FINAL RESULT
Used togenerate output
BEST PRACTICES
Use a combination of semanticsearch and deterministic querieswhere possible
+ Cache output where possible
COMMON PITFALLS
➔ The wrong elements could becompared when looking attext similarity, that is whyre-ranking is important

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval: search
How to search?
Semantic search
Keyword search
Search query
There are many diﬀerentapproaches to search depending onthe use case and the existingsystem.
Using embeddings, youcan perform semanticsearches. You cancompare embeddingswith what is in yourdatabase and ﬁnd 
themost similar.
If you have extractedspeciﬁc entities orkeywords to search for,you can search for thesein your database.
Based on the extractedentities you have or theuser input as is, you canconstruct search <span style="color: #800080; text-decoration-color: #800080; font-weight: bold">queries</span><span style="font-weight: bold">(</span>SQL, cypher…<span style="font-weight: bold">)</span> and 
usethese queries to searchyour database.
You can use a hybrid approach and combine several of these.
You can perform multiple searches in parallel or in sequence, orsearch for keywords with their embeddings for 
example.

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval: multi-step retrieval
What is multi-step retrieval?
In some cases, there might beseveral actions to be performed toget the required information togenerate an answer.
Things to consider
●
Framework to be used:
○ When there are multiple steps to perform,consider whether you want to handle thisyourself or use a framework to 
make it easier
●
Cost &#x26; Latency:
○
○
Performing multiple steps at the retrievalstage can increase latency and costsigniﬁcantly
Consider performing actions in parallel toreduce latency
●
Chain of Thought:
○
○
Guide the assistant with the chain of thoughtapproach: break down instructions intoseveral steps, with clear 
guidelines onwhether to continue, stop or do somethingelse.This is more appropriate when tasks need tobe performed 
sequentially - for example: “ifthis didn’t work, then do this”

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Retrieval: re-ranking
What is re-ranking?
Example approaches
Re-ranking means re-ordering theresults of the retrieval process tosurface more relevant results.
This is particularly important whendoing semantic searches.
Rule-based re-ranking
You can use metadata to rank results by relevance. Forexample, you can look at the recency of the documents, 
attags, speciﬁc keywords in the title, etc.
Re-ranking algorithms
There are several existing algorithms/approaches you can usebased on your use case: BERT-based 
re-rankers,cross-encoder re-ranking, TF-IDF algorithms…

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation
FINAL RESULT
Piece of contentretrieved
LLM
Prompt includingthe content
User sees theﬁnal result
BEST PRACTICES
Evaluate performance after eachexperimentation to assess if it’sworth exploring other paths
+ Implement guardrails if applicable
COMMON PITFALLS
➔ Going for ﬁne-tuning withouttrying other approaches
➔ Not paying attention to theway the model is prompted

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation: context window
How to manage context?
Depending on your use case, there areseveral things to consider whenincluding retrieved content into thecontext 
window to generate an answer.
Things to consider
●
Context window max size:
○
○
There is a maximum size, so putting toomuch content is not ideal
In conversation use cases, theconversation will be part of the contextas well and will add to that size
●
Cost &#x26; Latency vs Accuracy:
○ More context results in increased
latency and additional costs since therewill be more input tokens
Less context might also result indecreased accuracy
○
●
“Lost in the middle” problem:
○ When there is too much context, LLMstend to forget the text “in the middle” ofthe content and might look over 
someimportant information.

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation: optimisation
How to optimise?
There are a few diﬀerentmethods to consider whenoptimising a RAG application.
Try them from left to right, anditerate with several of theseapproaches if needed.
Prompt Engineering
Few-shot examples
Fine-tuning
At each point of theprocess, experiment withdiﬀerent prompts to getthe expected input formator generate a 
relevantoutput.
Try guiding the model ifthe process to get to theﬁnal outcome containsseveral steps.
If the model doesn’tbehave as expected,provide examples of whatyou want e.g. provideexample user inputs andthe 
expected processingformat.
If giving a few examplesisn’t enough, considerﬁne-tuning a model withmore examples for eachstep of the process: 
youcan ﬁne-tune to get aspeciﬁc input processingor output format.

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Answer Generation: safety checks
Why include safety checks?
Just because you provide the modelwith <span style="font-weight: bold">(</span>supposedly<span style="font-weight: bold">)</span> relevant contextdoesn’t mean the answer willsystematically be 
truthful or on-point.
Depending on the use case, youmight want to double-check.
Example evaluation framework: RAGAS

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"></pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Overview**
Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span> enhances language models by integrating them with a retrieval system. This 
combination allows the model to access external knowledge sources, resulting in more accurate and contextually 
relevant responses.
**Example Use Cases:**
- Providing answers with up-to-date information
- Generating contextual responses
**What We’ll Cover:**
- Technical patterns
- Best practices
- Common pitfalls
- Resources
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns**
This image outlines four key technical patterns involved in data processing and answer generation:
. **Data Preparation**
   - **Chunking**: Breaking down data into smaller, manageable pieces.
   - **Embeddings**: Converting data into numerical formats that can be easily processed by machine learning 
models.
   - **Augmenting Content**: Enhancing data with additional information to improve its quality or usefulness.
. **Input Processing**
   - **Input Augmentation**: Adding extra data or features to the input to improve model performance.
   - **NER <span style="font-weight: bold">(</span>Named Entity Recognition<span style="font-weight: bold">)</span>**: Identifying and classifying key entities in the text, such as names, 
dates, and locations.
   - **Embeddings**: Similar to data preparation, embeddings are used here to represent input data in a format 
suitable for processing.
. **Retrieval**
   - **Search**: Locating relevant information from a dataset.
   - **Multi-step Retrieval**: Using multiple steps or methods to refine the search process and improve accuracy.
   - **Re-ranking**: Adjusting the order of retrieved results based on relevance or other criteria.
. **Answer Generation**
   - **Context Window**: Using a specific portion of data to generate relevant answers.
   - **Optimisation**: Improving the efficiency and accuracy of the answer generation process.
   - **Safety Checks**: Ensuring that the generated answers are safe and appropriate for use.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Data Preparation**
This presentation focuses on the process of preparing data for easier consumption by large language models <span style="font-weight: bold">(</span>LLMs<span style="font-weight: bold">)</span>.
. **Content Chunking**:   - Documents are divided into smaller, manageable pieces. This makes it easier for LLMs to
process the information.
. **Embeddings**:
   - Each chunk of content is converted into embeddings, which are numerical representations <span style="font-weight: bold">(</span>e.g., <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span><span style="font-weight: bold">)</span> that capture the semantic meaning of the text. These embeddings are then stored in a knowledge base.
. **Augmenting Content**:
   - Content can be enhanced using LLMs. For example, GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can be used to rephrase, summarize, and generate bullet
points from the text.
. **Best Practices**:
   - Pre-process content for LLM consumption by adding summaries and headers for each part.
   - Curate relevant data sources to ensure quality and relevance.
. **Common Pitfalls**:
   - Avoid having too much low-quality content.
   - Ensure documents are not too large, as this can hinder processing efficiency.
This approach helps in organizing and optimizing data for better performance and understanding by LLMs.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Data Preparation - Chunking**
**Why Chunking?**
Chunking is a technique used when your system doesn't need entire documents to provide relevant answers. By 
breaking documents into smaller pieces, you can make data easier to process, which reduces cost and latency. This 
approach is beneficial for systems that need to handle large volumes of data efficiently. Other methods for data 
preparation include using graphs or map-reduce.
**Things to Consider**
. **Overlap:**
   - Should chunks be independent or overlap with one another?
   - If they overlap, by how much should they do so?
. **Size of Chunks:**
   - What is the optimal chunk size for your specific use case?
   - Do you want to include a lot of information in the context window, or just the minimum necessary?
. **Where to Chunk:**
   - Should you chunk every N tokens or use specific separators?
   - Is there a logical way to split the context that would aid the retrieval process?
. **What to Return:**
   - Should you return chunks across multiple documents or focus on top chunks within the same document?
   - Should chunks be linked together with metadata to indicate common properties?
These considerations help in designing an efficient chunking strategy that aligns with your system's requirements 
and goals.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"># Technical Patterns: Data Preparation - Embeddings
## What to Embed?
When preparing data for embedding, it's important to consider not just the text but also the metadata. This 
approach can enhance the searchability and relevance of the data. Here are some examples:
### Examples
. **Embedding Q&#x26;A Posts in a Forum**
   - You might want to include the title of the posts, the original question, and the top answers.
   - Additionally, if the posts are tagged by topic or keywords, these can be embedded as well.
. **Embedding Product Specs**
   - Besides embedding the text from product descriptions, you can add metadata such as color, size, and other 
specifications to your embeddings.
By embedding both text and metadata, you can improve the ability to surface specific chunks or documents during a 
search.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Data Preparation - Augmenting Content**
**What does “Augmenting content” mean?**
Augmenting content involves modifying the original material to make it more accessible and understandable for 
systems that rely on Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span>. These modifications can include changes in format, 
wording, or the addition of descriptive elements like summaries or keywords.
**Example Approaches:**
. **Make it a Guide:**
   - Reformat the content into a step-by-step guide with clear headings and bullet points. This structure is more 
easily understood by a Language Learning Model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span>. GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can assist with this transformation using the right 
prompts.
. **Add Descriptive Metadata:**
   - Incorporate keywords or text that users might search for when considering a specific product or service. This 
helps in making the content more searchable and relevant.
. **Multimodality:**
   - Utilize models like Whisper or GPT-4V to convert audio or visual content into text. For instance, GPT-4V can 
generate tags for images or describe slides, enhancing the content's accessibility and utility.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Input Processing**
 methods for processing input data according to specific tasks, focusing on three main areas: Q&#x26;A, content search, 
and database <span style="font-weight: bold">(</span>DB<span style="font-weight: bold">)</span> search.
. **Q&#x26;A**:   - Uses a technique called HyDE, where a large language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> is asked to hypothetically answer a
question. This answer is then used to search the knowledge base <span style="font-weight: bold">(</span>KB<span style="font-weight: bold">)</span>.
. **Content Search**:
   - Involves prompting the LLM to rephrase the input and optionally add more context to improve search results.
. **DB Search**:
   - Utilizes Named Entity Recognition <span style="font-weight: bold">(</span>NER<span style="font-weight: bold">)</span> to find relevant entities. These entities are then used for keyword 
searches or to construct a search query.
 highlights different output formats:
- **Embeddings**: Numerical representations of data, such as vectors <span style="font-weight: bold">(</span>e.g., <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span><span style="font-weight: bold">)</span>.
- **Query**: SQL-like statements for database searches <span style="font-weight: bold">(</span>e.g., SELECT * from items<span style="font-weight: bold">)</span>.
- **Keywords**: Specific terms extracted from the input <span style="font-weight: bold">(</span>e.g., <span style="color: #008000; text-decoration-color: #008000">"red,"</span> <span style="color: #008000; text-decoration-color: #008000">"summer"</span><span style="font-weight: bold">)</span>.
**Best Practices**:
- Transform the input to match the content in the database.
- Use metadata to enhance user input.
**Common Pitfalls**:
- Avoid directly comparing input to the database without considering the specific requirements of the task.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Input Processing - Input Augmentation**
**What is input augmentation?**
Input augmentation involves transforming the input into something different, such as rephrasing it, splitting it 
into several inputs, or expanding it. This process enhances performance by helping the language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> better 
understand the user's intent.
**Example Approaches:**
. **Query Expansion**
   - Rephrase the query to make it more descriptive. This helps the LLM grasp the context and details more 
effectively.
. **HyDE**
   - Hypothetically answer the question and use that answer to search the knowledge base <span style="font-weight: bold">(</span>KB<span style="font-weight: bold">)</span>. This approach can 
provide more relevant results by anticipating possible answers.
. **Splitting a Query in N**
   - When a user query contains multiple questions or intents, consider dividing it into several queries. This 
ensures each part is addressed thoroughly.
. **Fallback**
   - Implement a flow where the LLM can ask for clarification if the original query lacks sufficient information. 
This is particularly useful when using tools that require precise input.
*Note: GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> can perform these tasks with the appropriate prompt.*
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Input Processing - NER
**Why use NER?**
Named Entity Recognition <span style="font-weight: bold">(</span>NER<span style="font-weight: bold">)</span> is a technique used to extract relevant entities from input data. This process is 
beneficial for creating more deterministic search queries, especially when the scope is very constrained. By 
identifying specific entities, such as names, dates, or locations, NER helps in refining and improving the accuracy
of searches.
**Example: Searching for Movies**
Consider a structured database containing metadata on movies. By using NER, you can extract specific entities like 
genre, actors, or directors' names from a user's query. This information can then be used to search the database 
more effectively.
**Note:** After extracting the relevant entities, you can use exact values or embeddings to enhance the search 
process.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Retrieval
This diagram illustrates a retrieval process using technical patterns. The process begins with three types of 
input: embeddings, queries, and keywords.
. **Embeddings**: These are numerical representations <span style="font-weight: bold">(</span>e.g., <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.983</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.123</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.289</span><span style="font-weight: bold">)</span> used for semantic search. They 
are processed through a vector database <span style="font-weight: bold">(</span>vector DB<span style="font-weight: bold">)</span>.
. **Query**: This involves structured queries <span style="font-weight: bold">(</span>e.g., <span style="color: #008000; text-decoration-color: #008000">"SELECT * from items..."</span><span style="font-weight: bold">)</span> that interact with a relational or 
NoSQL database.
. **Keywords**: Simple search terms like <span style="color: #008000; text-decoration-color: #008000">"red"</span> and <span style="color: #008000; text-decoration-color: #008000">"summer"</span> are also used with the relational or NoSQL database.
The results from both the vector and relational/NoSQL databases are combined. The initial results undergo a 
re-ranking process to ensure accuracy and relevance, leading to the final result, which is then used to generate 
output.
**Best Practices**:
- Combine semantic search with deterministic queries for more effective retrieval.
- Cache outputs where possible to improve efficiency.
**Common Pitfalls**:
- Incorrect element comparison during text similarity checks can occur, highlighting the importance of re-ranking 
to ensure accurate results.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Retrieval - Search
**How to search?**
There are various approaches to searching, which depend on the use case and the existing system. Here are three 
main methods:
. **Semantic Search**:
   - This method uses embeddings to perform searches.   - By comparing embeddings with the data in your database, 
you can find the most similar matches.
. **Keyword Search**:
   - If you have specific entities or keywords extracted, you can search for these directly in your database.
. **Search Query**:
   - Based on extracted entities or direct user input, you can construct search queries <span style="font-weight: bold">(</span>such as SQL or Cypher<span style="font-weight: bold">)</span> to 
search your database.
Additionally, you can use a hybrid approach by combining several methods. This can involve performing multiple 
searches in parallel or in sequence, or searching for keywords along with their embeddings.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Retrieval - Multi-step Retrieval**
**What is multi-step retrieval?**
Multi-step retrieval involves performing several actions to obtain the necessary information to generate an answer.
This approach is useful when a single step is insufficient to gather all required data.
**Things to Consider**
. **Framework to be Used:**
   - When multiple steps are needed, decide whether to manage this process yourself or use a framework to simplify 
the task.
. **Cost &#x26; Latency:**
   - Performing multiple steps can significantly increase both latency and cost.
   - To mitigate latency, consider executing actions in parallel.
. **Chain of Thought:**
   - Use a chain of thought approach to guide the process. Break down instructions into clear steps, providing 
guidelines on whether to continue, stop, or take alternative actions.
   - This method is particularly useful for tasks that must be performed sequentially, such as <span style="color: #008000; text-decoration-color: #008000">"if this didn’t </span>
<span style="color: #008000; text-decoration-color: #008000">work, then do this."</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Retrieval - Re-ranking**
**What is re-ranking?**
Re-ranking involves re-ordering the results of a retrieval process to highlight more relevant outcomes. This is 
especially crucial in semantic searches, where understanding the context and meaning of queries is important.
**Example Approaches**
. **Rule-based Re-ranking**
   - This approach uses metadata to rank results by relevance. For instance, you might consider the recency of 
documents, tags, or specific keywords in the title to determine their importance.
. **Re-ranking Algorithms**
   - There are various algorithms available for re-ranking based on specific use cases. Examples include BERT-based
re-rankers, cross-encoder re-ranking, and TF-IDF algorithms. These methods apply different techniques to assess and
order the relevance of search results.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Answer Generation**
This diagram illustrates the process of generating answers using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span>. Here's a breakdown of the 
components and concepts:
. **Process Flow:**
   - A piece of content is retrieved and used to create a prompt.
   - This prompt is fed into the LLM, which processes it to generate a final result.
   - The user then sees this final result.
. **Best Practices:**
   - It's important to evaluate performance after each experiment. This helps determine if exploring other methods 
is beneficial.
   - Implementing guardrails can be useful to ensure the model's outputs are safe and reliable.
. **Common Pitfalls:**
   - Avoid jumping straight to fine-tuning the model without considering other approaches that might be more 
effective or efficient.
   - Pay close attention to how the model is prompted, as this can significantly impact the quality of the output.
By following these guidelines, you can optimize the use of LLMs for generating accurate and useful answers.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"># Technical Patterns: Answer Generation - Context Window
## How to Manage Context?
When generating answers using a context window, it's important to consider several factors based on your specific 
use case. Here are key points to keep in mind:
### Things to Consider
- **Context Window Max Size:**
  - The context window has a maximum size, so overloading it with too much content is not ideal.
  - In conversational scenarios, the conversation itself becomes part of the context, contributing to the overall 
size.
- **Cost &#x26; Latency vs. Accuracy:**
  - Including more context can lead to increased latency and higher costs due to the additional input tokens 
required.
  - Conversely, using less context might reduce accuracy.
- **<span style="color: #008000; text-decoration-color: #008000">"Lost in the Middle"</span> Problem:**
  - When the context is too extensive, language models may overlook or forget information that is <span style="color: #008000; text-decoration-color: #008000">"in the middle"</span> 
of the content, potentially missing important details.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Answer Generation Optimisation**
**How to optimise?**
When optimising a Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span> application, there are several methods to consider. These 
methods should be tried sequentially from left to right, and multiple approaches can be iterated if necessary.
. **Prompt Engineering**
   - Experiment with different prompts at each stage of the process to achieve the desired input format or generate
relevant output.
   - Guide the model through multiple steps to reach the final outcome.
. **Few-shot Examples**
   - If the model's behavior is not as expected, provide examples of the desired outcome.
   - Include sample user inputs and the expected processing format to guide the model.
. **Fine-tuning**
   - If a few examples are insufficient, consider fine-tuning the model with more examples for each process step.
   - Fine-tuning can help achieve a specific input processing or output format.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Answer Generation - Safety Checks
**Why include safety checks?**
Safety checks are crucial because providing a model with supposedly relevant context does not guarantee that the 
generated answer will be truthful or accurate. Depending on the use case, it is important to double-check the 
information to ensure reliability.
**RAGAS Score Evaluation Framework**
The RAGAS score is an evaluation framework that assesses both the generation and retrieval aspects of answer 
generation:
- **Generation:**
  - **Faithfulness:** This measures how factually accurate the generated answer is.
  - **Answer Relevancy:** This evaluates how relevant the generated answer is to the question.
- **Retrieval:**
  - **Context Precision:** This assesses the signal-to-noise ratio of the retrieved context, ensuring that the 
information is precise.
  - **Context Recall:** This checks if all relevant information required to answer the question is retrieved.
By using this framework, one can systematically evaluate and improve the quality of generated answers.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo ,  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> , and  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview  point to the latest model
version. You can verify this by looking at the response object after sending a request.
The response will include the specific model version used <span style="font-weight: bold">(</span>e.g.  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span> <span style="font-weight: bold">)</span>.
We also offer static model versions that developers can continue using for at least
three months after an updated model has been introduced. With the new cadence of
model updates, we are also giving people the ability to contribute evals to help us
improve the model for different use cases. If you are interested, check out the OpenAI
Evals repository.
Learn more about model deprecation on our deprecation page.
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is a large multimodal model <span style="font-weight: bold">(</span>accepting text or image inputs and outputting text<span style="font-weight: bold">)</span>
that can solve difficult problems with greater accuracy than any of our previous
models, thanks to its broader general knowledge and advanced reasoning capabilities.
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is available in the OpenAI API to paying customers. Like  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo , GPT-
 is optimized for chat but works well for traditional completions tasks using the Chat
Completions API. Learn how to use GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> in our text generation guide.
MODEL
DE S CRIPTION
CONTEXT
WIND OW
TRAINING
DATA
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview
New  GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
Up to
Dec
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>
The latest GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> model
tokens
intended to reduce cases of
“laziness” where the model
doesn’t complete a task.
Returns a maximum of
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
Learn more.
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview
Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">25</span>-preview.
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-preview
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo model
featuring improved
instruction following, JSON
mode, reproducible outputs,
parallel function calling, and
more. Returns a maximum
of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens. This
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens
Up to
Dec
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens
Up to
Apr <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
MODEL
DE S CRIPTION
is a preview model.
Learn more.
CONTEXT
WIND OW
TRAINING
DATA
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> with the ability to
understand images, in
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens
Up to
Apr <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
addition to all other GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
Turbo capabilities. Currently
points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-
vision-preview.
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-vision-preview GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> with the ability to
understand images, in
addition to all other GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
Turbo capabilities. Returns a
maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output
tokens. This is a preview
model version. Learn more.
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span>
tokens
Up to
Apr <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>
Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span>
Up to
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>. See
tokens
Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
continuous model upgrades.
Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> from
June 13th <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> with
improved function calling
support.
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span>
tokens
Up to
Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k
Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>
k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. See
continuous model upgrades.
This model was never rolled
out widely in favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
Turbo.
Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k
from June 13th <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> with
improved function calling
support. This model was
never rolled out widely in
favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo.
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>
tokens
Up to
Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>
tokens
Up to
Sep <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
For many basic tasks, the difference between GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> models is not
significant. However, in more complex reasoning situations, GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is much more
capable than any of our previous models.
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
Multilingual capabilities
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> outperforms both previous large language models and as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, most state-
of-the-art systems <span style="font-weight: bold">(</span>which often have benchmark-specific training or hand-
engineering<span style="font-weight: bold">)</span>. On the MMLU benchmark, an English-language suite of multiple-choice
questions covering <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">57</span> subjects, GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> not only outperforms existing models by a
considerable margin in English, but also demonstrates strong performance in other
languages.
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo models can understand and generate natural language or code and
have been optimized for chat using the Chat Completions API but work well for non-
chat tasks as well.
CONTEXT
WIND OW
TRAINING
DATA
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>
tokens
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
MODEL
DE S CRIPTION
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>
New  Updated GPT <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo
The latest GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo
model with higher accuracy at
responding in requested
formats and a fix for a bug
which caused a text encoding
issue for non-English
language function calls.
Returns a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
output tokens. Learn more.
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
Up to Sep
turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. The gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-
tokens
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
turbo model alias will be
automatically upgraded from
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> to
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span> on
February 16th.
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo model with
improved instruction
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>
tokens
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
following, JSON mode,
reproducible outputs, parallel
function calling, and more.
Returns a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
output tokens. Learn more.
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
MODEL
DE S CRIPTION
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct Similar capabilities as GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>
era models. Compatible with
legacy Completions endpoint
and not Chat Completions.
CONTEXT
WIND OW
TRAINING
DATA
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
tokens
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k
Legacy  Currently points to
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>.
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>
tokens
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>
Legacy  Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-
turbo from June 13th <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.
Will be deprecated on June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>,
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">24</span>.
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span>
tokens
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>
Legacy  Snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span>
Up to Sep
k-turbo from June 13th
tokens
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>. Will be deprecated on
June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.
DALL·E
DALL·E is a AI system that can create realistic images and art from a description in
natural language. DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> currently supports the ability, given a prompt, to create a
new image with a specific size. DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> also support the ability to edit an existing
image, or create variations of a user provided image.
DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> is available through our Images API along with DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. You can try DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>
through ChatGPT Plus.
MODEL
DE S CRIPTION
dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>
New  DALL·E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>
The latest DALL·E model released in Nov <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Learn more.
dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> The previous DALL·E model released in Nov <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2022</span>. The 2nd iteration of
DALL·E with more realistic, accurate, and 4x greater resolution images
than the original model.
TTS
TTS is an AI model that converts text to natural sounding spoken text. We offer two
different model variates,  tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>  is optimized for real time text to speech use cases
and  tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd  is optimized for quality. These models can be used with the Speech
endpoint in the Audio API.
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
MODEL
DE S CRIPTION
tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>
New  Text-to-speech <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>
The latest text to speech model, optimized for speed.
tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd
New  Text-to-speech <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> HD
The latest text to speech model, optimized for quality.
Whisper
Whisper is a general-purpose speech recognition model. It is trained on a large dataset
of diverse audio and is also a multi-task model that can perform multilingual speech
recognition as well as speech translation and language identification. The Whisper v2-
large model is currently available through our API with the  whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>  model name.
Currently, there is no difference between the open source version of Whisper and the
version available through our API. However, through our API, we offer an optimized
inference process which makes running Whisper through our API much faster than
doing it through other means. For more technical details on Whisper, you can read the
paper.
Embeddings
Embeddings are a numerical representation of text that can be used to measure the
relatedness between two pieces of text. Embeddings are useful for search, clustering,
recommendations, anomaly detection, and classification tasks. You can read more
about our latest embedding models in the announcement blog post.
MODEL
DE S CRIPTION
text-embedding-
-large
New  Embedding V3 large
Most capable embedding model for both
english and non-english tasks
text-embedding-
New  Embedding V3 small
-small
Increased performance over 2nd generation ada
embedding model
text-embedding-
ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>
Most capable 2nd generation embedding
model, replacing <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span> first generation models
OUTP UT
DIMENSION
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">072</span>
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>
Moderation
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
The Moderation models are designed to check whether content complies with
OpenAI's usage policies. The models provide classification capabilities that look for
content in the following categories: hate, hate/threatening, self-harm, sexual,
sexual/minors, violence, and violence/graphic. You can find out more in our moderation
guide.
Moderation models take in an arbitrary sized input that is automatically broken up into
chunks of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens. In cases where the input is more than <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens,
truncation is used which in a rare condition may omit a small number of tokens from
the moderation check.
The final results from each request to the moderation endpoint shows the maximum
value on a per category basis. For example, if one chunk of 4K tokens had a category
score of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span> and the other had a score of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1901</span>, the results would show <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span> in the
API response since it is higher.
MODEL
DE S CRIPTION
MAX
TOKENS
text-moderation-latest Currently points to text-moderation-
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">7</span>.
text-moderation-stable Currently points to text-moderation-
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">7</span>.
text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span>
Most capable moderation model across
all categories.
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span>
GPT base
GPT base models can understand and generate natural language or code but are not
trained with instruction following. These models are made to be replacements for our
original GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> base models and use the legacy Completions API. Most customers
should use GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> or GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>.
MODEL
DE S CRIPTION
babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span> Replacement for the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> ada and
babbage base models.
davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span> Replacement for the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> curie and
davinci base models.
MAX
TOKENS
TRAINING
DATA
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span>
tokens
,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span>
tokens
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
Up to Sep
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">21</span>
How we use your data
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
Your data is your data.
As of March <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, data sent to the OpenAI API will not be used to train or improve
OpenAI models <span style="font-weight: bold">(</span>unless you explicitly opt in<span style="font-weight: bold">)</span>. One advantage to opting in is that the
models may get better at your use case over time.
To help identify abuse, API data may be retained for up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, after which it will be
deleted <span style="font-weight: bold">(</span>unless otherwise required by law<span style="font-weight: bold">)</span>. For trusted customers with sensitive
applications, zero data retention may be available. With zero data retention, request
and response bodies are not persisted to any logging mechanism and exist only in
memory in order to serve the request.
Note that this data policy does not apply to OpenAI's non-API consumer services like
ChatGPT or DALL·E Labs.
Default usage policies by endpoint
ENDP OINT
DATA USED
FOR TRAINING
DEFAULT
RETENTION
ELIGIBLE FOR
ZERO RETENTION
<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>*
No
 days
Yes, except
image inputs*
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">files</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">threads</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">messages</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">runs</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/threads/runs/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">steps</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">generations</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">edits</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">variations</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>
No
No
No
No
No
No
No
No
No
No
<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span> No
Until deleted by
No
customer
Until deleted by
No
customer
 days *
 days *
 days *
 days *
 days
 days
 days
 days
Zero data
retention
No
No
No
No
No
No
No
Yes
-
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
Models - OpenAI API
ENDP OINT
DATA USED
FOR TRAINING
DEFAULT
RETENTION
ELIGIBLE FOR
ZERO RETENTION
<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>
No
<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>
No
No
No
No
Zero data
retention
 days
Until deleted by
customer
Zero data
retention
-
No
No
-
 days
Yes
* Image inputs via the  gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview  model are not eligible for zero
retention.
* For the Assistants API, we are still evaluating the default retention period during the
Beta. We expect that the default retention period will be stable after the end of the
Beta.
For details, see our API data usage policies. To learn more about zero retention, get in
touch with our sales team.
Model endpoint compatibility
ENDP OINT
L ATE ST MODEL S
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>
All models except gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0301</span>
supported. The retrieval tool requires gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-
turbo-preview <span style="font-weight: bold">(</span>and subsequent dated model
releases<span style="font-weight: bold">)</span> or gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span> <span style="font-weight: bold">(</span>and
subsequent versions<span style="font-weight: bold">)</span>.
<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span> whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>
whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>
tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd
<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and dated model releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-
preview and dated model releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-
vision-preview, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k and dated model
releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo and dated model
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">26</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">02</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>, <span style="color: #00ff00; text-decoration-color: #00ff00; font-weight: bold">17:58</span>
ENDP OINT
Models - OpenAI API
L ATE ST MODEL S
releases, gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k and dated model
releases, fine-tuned versions of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span> <span style="font-weight: bold">(</span>Legacy<span style="font-weight: bold">)</span> gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct, babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>,
davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>
text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small, text-embedding-
-large, text-embedding-ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo, babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>, davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>
<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>
text-moderation-stable, text-
<span style="color: #0000ff; text-decoration-color: #0000ff; text-decoration: underline">https://platform.openai.com/docs/models/overview</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">10</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"></pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo**
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is a sophisticated multimodal model capable of processing both text and image inputs to produce text outputs.
It is designed to tackle complex problems with higher accuracy than previous models, leveraging its extensive 
general knowledge and advanced reasoning skills. GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> is accessible through the OpenAI API for paying customers 
and is optimized for chat applications, although it can also handle traditional completion tasks using the Chat 
Completions API.
**Model Versions:**
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview**
   - **Description:** This is the latest GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo model, designed to minimize instances where the model fails to
complete a task, known as <span style="color: #008000; text-decoration-color: #008000">"laziness."</span> It can return up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data:** Up to December <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview**
   - **Description:** This version currently points to the gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>-preview model.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data:** Up to December <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-preview**
   - **Description:** This version of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo includes enhancements such as improved instruction following, 
JSON mode, reproducible outputs, and parallel function calling. It also supports up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data:** Up to April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
These models are part of OpenAI's ongoing efforts to provide developers with robust tools for various applications,
ensuring flexibility and improved performance across different use cases.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Models - OpenAI API Overview**
This document provides an overview of various GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> models, highlighting their capabilities, context windows, and 
training data timelines.
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview**
   - **Description**: This model has the ability to understand images, in addition to all other GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo 
capabilities. It currently points to the gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-vision-preview model.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data**: Up to April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>-vision-preview**
   - **Description**: Similar to the gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview, this model can understand images and includes all GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> 
Turbo capabilities. It returns a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens and is a preview model version.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">128</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">000</span> tokens
   - **Training Data**: Up to April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>**
   - **Description**: This model currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> and includes continuous model upgrades.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span> tokens
   - **Training Data**: Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description**: A snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> from June 13th, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, with improved function calling support.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">192</span> tokens
   - **Training Data**: Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k**
   - **Description**: This model points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span> and includes continuous model upgrades. It was not widely
rolled out in favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens
   - **Training Data**: Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description**: A snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k from June 13th, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, with improved function calling support. Like 
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k, it was not widely rolled out in favor of GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> Turbo.
   - **Context Window**: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens
   - **Training Data**: Up to September
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Multilingual Capabilities and GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo**
**Multilingual Capabilities**
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> surpasses previous large language models and, as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, most state-of-the-art systems. It excels in the 
MMLU benchmark, which involves English-language multiple-choice questions across <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">57</span> subjects. GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> not only 
outperforms existing models in English but also shows strong performance in other languages.
**GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo**
GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo models are designed to understand and generate natural language or code. They are optimized for chat 
using the Chat Completions API but are also effective for non-chat tasks.
**Model Descriptions:**
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span>**
   - **Description:** Updated GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Turbo with improved accuracy and a fix for a text encoding bug in non-English
language function calls. It returns up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo**
   - **Description:** Currently points to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>. The alias will automatically upgrade to 
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0125</span> on February 16th.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>**
   - **Description:** Features improved instruction following, JSON mode, reproducible outputs, and parallel 
function calling. It returns up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> output tokens.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Models - OpenAI API**
**GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> Models:**
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct**
   - **Description:** Similar capabilities to GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> era models. Compatible with legacy Completions endpoint, not 
Chat Completions.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k**
   - **Description:** Legacy model pointing to gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description:** Legacy snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo from June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Will be deprecated on June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
. **gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0613</span>**
   - **Description:** Legacy snapshot of gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k-turbo from June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>. Will be deprecated on June <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">13</span>,
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2024</span>.
   - **Context Window:** <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">385</span> tokens
   - **Training Data:** Up to September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>
**DALL-E:**
- DALL-E is an AI system that creates realistic images and art from natural language descriptions. DALL-E <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> 
supports creating new images with specific sizes and editing existing images or creating variations. Available 
through the Images API and ChatGPT Plus.
. **dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>**
   - **Description:** The latest DALL-E model released in November <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.
. **dall-e-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>**
   - **Description:** Released in November <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2022</span>, this model offers more realistic, accurate, and higher resolution 
images than the original.
**TTS <span style="font-weight: bold">(</span>Text-to-Speech<span style="font-weight: bold">)</span>:**
- TTS converts text to natural-sounding spoken text. Two model variants are offered:
  - **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>:** Optimized for real-time text-to-speech use cases.
  - **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd:** Optimized for quality.
- These models can be used with the Speech endpoint in
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Models - OpenAI API**
**Text-to-Speech Models:**
. **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>**: This is a new text-to-speech model optimized for speed, providing efficient conversion of text into 
spoken words.
  <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>. **tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd**: This model is optimized for quality, offering high-definition text-to-speech conversion.
**Whisper:**
Whisper is a versatile speech recognition model capable of handling diverse audio inputs. It supports multilingual 
speech recognition, speech translation, and language identification. The Whisper v2-large model is accessible via 
the API under the name <span style="color: #008000; text-decoration-color: #008000">"whisper-1."</span> While the open-source version and the API version are similar, the API offers 
an optimized inference process for faster performance. More technical details can be found in the associated paper.
**Embeddings:**
Embeddings are numerical representations of text, useful for measuring the relatedness between text pieces. They 
are applied in search, clustering, recommendations, anomaly detection, and classification tasks.
- **text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-large**: The most capable embedding model for both English and non-English tasks, with an 
output dimension of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">072</span>.
 - **text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small**: Offers improved performance over the second-generation ada embedding model, with an 
output dimension of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>.
 - **text-embedding-ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>**: A second-generation embedding model replacing <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span> first-generation models, also with 
an output dimension of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">536</span>.
**Moderation:**
The document mentions a section on moderation, likely related to content moderation capabilities, though specific 
details are not provided in the visible content.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Moderation Models and GPT Base**
**Moderation Models**
The moderation models are designed to ensure content compliance with OpenAI's usage policies. They classify content
into categories such as hate, hate/threatening, self-harm, sexual, sexual/minors, violence, and violence/graphic. 
These models process inputs by breaking them into chunks of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">096</span> tokens. If the input exceeds <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens, some 
tokens may be truncated, potentially omitting a few from the moderation check.
The moderation endpoint provides the maximum score per category from each request. For instance, if one chunk 
scores <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span> and another scores <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1901</span> in a category, the API response will show <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.9901</span>.
- **text-moderation-latest**: Points to text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span> with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens.
- **text-moderation-stable**: Also points to text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span> with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens.
- **text-moderation-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">007</span>**: The most capable model across all categories with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">32</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">768</span> tokens.
**GPT Base**
GPT base models are capable of understanding and generating natural language or code but are not trained for 
instruction following. They serve as replacements for the original GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> base models and utilize the legacy 
Completions API. Most users are advised to use GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> or GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>.
- **babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>**: Replaces the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> ada and babbage models, with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span> tokens and training data up to 
September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>.
- **davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>**: Replaces the GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> curie and davinci models, with a max of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">16</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">384</span> tokens and training data up to
September <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2021</span>.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Your Data is Your Data
As of March <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>, data sent to the OpenAI API is not used to train or improve OpenAI models unless you 
explicitly opt in. Opting in can help models improve for your specific use case over time.
To prevent abuse, API data may be retained for up to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days before deletion, unless legally required otherwise. 
Trusted customers with sensitive applications may have zero data retention, meaning request and response bodies are
not logged and exist only in memory to serve the request.
This data policy does not apply to OpenAI's non-API consumer services like ChatGPT or DALL-E Labs.
**Default Usage Policies by Endpoint**
- **<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>**: Data is not used for training. Default retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, and it is eligible for 
zero retention except for image inputs.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">files</span>**: Data is not used for training. Retention is until deleted by the customer, with no zero retention 
option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>**: Data is not used for training. Retention is until deleted by the customer, with no zero 
retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">threads</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">messages</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/threads/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">runs</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/threads/runs/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">steps</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">60</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">generations</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">edits</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/images/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">variations</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, with no zero retention option.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>**: Data is not used for training. Retention is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days, and it is eligible for zero retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span>**: Data is not used for training
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">### Model Endpoint Compatibility and Data Retention
#### Data Retention Details
The table outlines the data retention policies for various API endpoints:
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>**: No data is used for training, and there is zero data retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>**: No data is used for training, with a default retention period of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days. It is not 
eligible for zero retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>**: No data is used for training, and data is retained until deleted by the customer. It is
not eligible for zero retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>**: No data is used for training, and there is zero data retention.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>**: No data is used for training, with a default retention period of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">30</span> days. It is eligible for
zero retention.
Additional notes:
- Image inputs via the `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview` model are not eligible for zero retention.
- The default retention period for the Assistants API is still being evaluated during the Beta phase.
#### Model Endpoint Compatibility
The table provides information on the compatibility of endpoints with the latest models:
- **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">assistants</span>**: Supports all models except `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0301</span>`. The `retrieval` tool requires 
`gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview` or `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1106</span>`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">transcriptions</span>**: Compatible with `whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">translations</span>**: Compatible with `whisper-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/audio/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">speech</span>**: Compatible with `tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>` and `tts-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-hd`.
- **<span style="color: #800080; text-decoration-color: #800080">/v1/chat/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span>**: Compatible with `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>`, `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-turbo-preview`, `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-vision-preview`, `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>-32k`, 
and `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo`.
For more details, users are encouraged to refer to the API data usage policies or contact the sales team for 
information on zero retention.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">LATEST MODELS
This document outlines the latest models available for different endpoints in the OpenAI API:
. **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">completions</span> <span style="font-weight: bold">(</span>Legacy<span style="font-weight: bold">)</span>**:
   - Models: `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-instruct`, `babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`, `davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`
   - These models are used for generating text completions based on input prompts.
. **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">embeddings</span>**:
   - Models: `text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-small`, `text-embedding-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>-large`, `text-embedding-ada-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`
   - These models are designed to convert text into numerical vectors, which can be used for various tasks like 
similarity comparison and clustering.
. **<span style="color: #800080; text-decoration-color: #800080">/v1/fine_tuning/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">jobs</span>**:
   - Models: `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo`, `babbage-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`, `davinci-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">002</span>`
   - These models support fine-tuning, allowing users to customize the models for specific tasks by training them 
on additional data.
. **<span style="color: #800080; text-decoration-color: #800080">/v1/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">moderations</span>**:
   - Models: `text-moderation-stable`
   - This model is used for content moderation, helping to identify and filter out inappropriate or harmful 
content.
Additionally, the document mentions the availability of `gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo-16k` and other fine-tuned versions of 
`gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo`, indicating enhancements in model capabilities and performance.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Overview
Evaluation is the process of validatingand testing the outputs that your LLMapplications are producing. 
Havingstrong evaluations <span style="font-weight: bold">(</span>“evals”<span style="font-weight: bold">)</span> will mean amore stable, reliable application which isresilient to code and model
changes.
Example use cases
- Quantify a solution’s reliability
- Monitor application performance in
production
Test for regressions
-
What we’ll cover
● What are evals
● Technical patterns
● Example framework
● Best practices
● Resources

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What are evals
Example
An evaluation contains a question and a correct answer. We call this the ground truth.
Question
What is the populationof Canada?
Thought: I don’t know. Ishould use a tool
Action: Search
Action Input: What is thepopulation of Canada?
LLM
Search
There are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> peoplein Canada as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.
The current population ofCanada is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> as ofTuesday, May <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>….
Actual result

An evaluation, or <span style="color: #008000; text-decoration-color: #008000">"eval,"</span> involves a question and a correct answer, known as the ground truth. In this example, the
question posed is, <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>
The process begins with a person asking this question. The language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> initially does not know the answer 
and decides to use a tool to find it. The LLM takes the action of searching, with the input being the question 
about Canada's population.
The search tool then provides the answer: <span style="color: #008000; text-decoration-color: #008000">"The current population of Canada is 39,566,248 as of Tuesday, May 23, </span>
<span style="color: #008000; text-decoration-color: #008000">2023."</span> This result matches the actual result expected, which is that there are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people in Canada as of 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.
This example illustrates how evaluations are used to verify the accuracy of information provided by a language 
model.
 an example of an evaluation process, often referred to as <span style="color: #008000; text-decoration-color: #008000">"evals."</span> The purpose of evals is to compare a predicted 
answer to a known correct answer, called the <span style="color: #008000; text-decoration-color: #008000">"ground truth,"</span> to determine if they match.
In this example, the question posed is: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span> The ground truth states that the 
population of Canada in <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people. The predicted answer is: <span style="color: #008000; text-decoration-color: #008000">"There are 39,566,248 people in Canada </span>
<span style="color: #008000; text-decoration-color: #008000">as of 2023."</span>
Since the predicted answer matches the ground truth, the evaluation is successful, as indicated by a checkmark. 
This process is crucial for verifying the accuracy of predictions in various applications.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What are evals
Example
Our ground truth matches the predicted answer, so the evaluation passes!
Evaluation
Question
Ground Truth
Predicted Answer
What is the populationof Canada?
The population of Canada in2023 is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people.
There are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> peoplein Canada as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.

An evaluation, or <span style="color: #008000; text-decoration-color: #008000">"eval,"</span> involves a question and a correct answer, known as the ground truth. In this example, the
question posed is, <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>
The process begins with a person asking this question. The language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> initially does not know the answer 
and decides to use a tool to find it. The LLM takes the action of searching, with the input being the question 
about Canada's population.
The search tool then provides the answer: <span style="color: #008000; text-decoration-color: #008000">"The current population of Canada is 39,566,248 as of Tuesday, May 23, </span>
<span style="color: #008000; text-decoration-color: #008000">2023."</span> This result matches the actual result expected, which is that there are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people in Canada as of 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.
This example illustrates how evaluations are used to verify the accuracy of information provided by a language 
model.
 an example of an evaluation process, often referred to as <span style="color: #008000; text-decoration-color: #008000">"evals."</span> The purpose of evals is to compare a predicted 
answer to a known correct answer, called the <span style="color: #008000; text-decoration-color: #008000">"ground truth,"</span> to determine if they match.
In this example, the question posed is: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span> The ground truth states that the 
population of Canada in <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> people. The predicted answer is: <span style="color: #008000; text-decoration-color: #008000">"There are 39,566,248 people in Canada </span>
<span style="color: #008000; text-decoration-color: #008000">as of 2023."</span>
Since the predicted answer matches the ground truth, the evaluation is successful, as indicated by a checkmark. 
This process is crucial for verifying the accuracy of predictions in various applications.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations
Component evaluations
Subjective evaluations
●
●
Comparison metrics likeBLEU, ROUGE
Gives a score to ﬁlter andrank results
●
●
Compares groundtruth to prediction
Gives Pass/Fail
●
●
Uses a scorecard toevaluate subjectively
Scorecard may alsohave a Pass/Fail

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations
ROUGE is a common metric for evaluating machine summarizations of text
ROUGE
Metric for evaluatingsummarization tasks
Original
OpenAI's mission is to ensure thatartiﬁcial general intelligence <span style="font-weight: bold">(</span>AGI<span style="font-weight: bold">)</span>beneﬁts all of humanity. OpenAIwill build 
safe and beneﬁcial AGIdirectly, but will also consider itsmission fulﬁlled if its work aidsothers to achieve this 
outcome.OpenAI follows several keyprinciples for this purpose. First,broadly distributed beneﬁts - anyinﬂuence over
AGI's deploymentwill be used for the beneﬁt of all,and to avoid harmful uses or undueconcentration of power…
MachineSummary
OpenAI aims to ensure AGI isfor everyone's use, totallyavoiding harmful stuff or bigpower concentration.Committed 
to researchingAGI's safe side, promotingthese studies in AI folks.OpenAI wants to be top in AIthings and works 
withworldwide research, policygroups to ﬁgure AGI's stuff.
ROUGEScore
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">51162</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations
BLEU score is another standard metric, this time focusing on machine translation tasks
BLEU
Original text
Reference
Translation
PredictedTranslation
Metric forevaluatingtranslation tasks
Y gwir oedddoedden nhwddim yn dweudcelwyddau wedi'rcwbl.
The truth wasthey were nottelling lies afterall.
The truth wasthey weren'ttelling lies afterall.
BLEUScore
.<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39938</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Metric-based evaluations
What they’re good for
What to be aware of
●
●
A good starting point for evaluating a
● Not tuned to your speciﬁc context
fresh solution
Useful yardstick for automated testing
of whether a change has triggered a
major performance shift
● Most customers require more
sophisticated evaluations to go to
production
● Cheap and fast

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Component evaluations
Component evaluations <span style="font-weight: bold">(</span>or “unit tests”<span style="font-weight: bold">)</span> cover a single input/output of the application. They checkwhether each 
component works in isolation, comparing the input to a ground truth ideal result
Is this thecorrect action?
Exact matchcomparison
Does this answeruse the context?
Extract numbersfrom each andcompare
What is the populationof Canada?
Thought: I don’t know. Ishould use a tool
Action: Search
Action Input: What is thepopulation of Canada?
Agent
Search
There are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> peoplein Canada as of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>.
The current population ofCanada is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">566</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">248</span> as ofTuesday, May <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">23</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span>….
Is this the rightsearch result?
Tag the rightanswer and doan exact matchcomparison withthe retrieval.

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical patterns
Subjective evaluations
Building up a good scorecard for automated testing beneﬁts from a few rounds of detailed humanreview so we can 
learn what is valuable.
A policy of “show rather than tell” is also advised for GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, so include examples of what a <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span> and8 out of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> 
look like so the model can appreciate the spread.
Examplescorecard
You are a helpful evaluation assistant who grades how well the Assistant has answered the customer’s query.
You will assess each submission against these metrics, please think through these step by step:
-
relevance: Grade how relevant the search content is to the question from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> <span style="color: #800080; text-decoration-color: #800080">//</span> <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> being highly relevant and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> 
beingnot relevant at all.
- credibility: Grade how credible the sources provided are from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> <span style="color: #800080; text-decoration-color: #800080">//</span> <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> being an established newspaper,
-
government agency or large company and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> being unreferenced.
result: Assess whether the question is correct given only the content returned from the search and the 
user’squestion <span style="color: #800080; text-decoration-color: #800080">//</span> acceptable values are “correct” or “incorrect”
You will output this as a JSON document: <span style="font-weight: bold">{</span>relevance: integer, credibility: integer, result: string<span style="font-weight: bold">}</span>
User: What is the population of Canada?
Assistant: Canada's population was estimated at <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">39</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">858</span>,<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">480</span> on April <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2023</span> by Statistics Canada.
Evaluation: <span style="font-weight: bold">{</span>relevance: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, credibility: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, result: correct<span style="font-weight: bold">}</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Example framework
Your evaluations can be grouped up into test suites called runs and executed in a batch to testthe eﬀectiveness of 
your system.
Each run should have its contents logged and stored at the most granular level <span style="color: #800080; text-decoration-color: #800080; font-weight: bold">possible</span><span style="font-weight: bold">(</span>“tracing”<span style="font-weight: bold">)</span> so you can 
investigate failure reasons, make tweaks and then rerun your evals.
Run ID Model
Score
Annotation feedback
Changes since last run





gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">28</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
<span style="color: #800080; text-decoration-color: #800080">/</span><span style="color: #ff00ff; text-decoration-color: #ff00ff">50</span>
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">34</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">18</span> incorrect with correct search results
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
N/A
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> incorrect with correct search results
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">12</span> incorrect with correct search results
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
Model updated to GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
Added few-shot examples
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">42</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> incorrect with correct search results
Added metadata to search
Prompt engineering for Answer step
gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">48</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
● <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> incorrect with correct search results
Prompt engineering to Answer step

This diagram illustrates a framework for processing a return request using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> system. Here's a 
breakdown of the process:
. **User Input**: The user wants to return a T-shirt purchased on Amazon on March 3rd.
. **Router**: The initial input is processed by a router LLM, which determines the nature of the request. The 
expected and predicted outcomes are both <span style="color: #008000; text-decoration-color: #008000">"return,"</span> and the process passes this evaluation.
. **Return Assistant**: The request is then handled by a return assistant LLM. It interacts with a knowledge base 
to verify the return policy.
. **Knowledge Base**: The system checks the return policy, confirming that the item is eligible for return within 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days of purchase. The expected and predicted outcomes are <span style="color: #008000; text-decoration-color: #008000">"return_policy,"</span> and this step also passes.
. **Response to User**: The system responds to the user, confirming that the return can be processed because it is 
within the <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>-day window.
. **Evaluation**: The response is evaluated for adherence to guidelines, scoring <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> for politeness, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for coherence,
and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for relevancy, resulting in a pass.
The framework uses both component evaluations <span style="font-weight: bold">(</span>red dashed lines<span style="font-weight: bold">)</span> and subjective evaluations <span style="font-weight: bold">(</span>orange dashed lines<span style="font-weight: bold">)</span> 
to ensure the process is accurate and user-friendly.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Example framework
I want to return aT-shirt I bought onAmazon on March 3rd.
User
Router
LLM
Expected: return
Predicted: return
PASS
Return
Assistant
LLM
Component evals
Subjective evals
Expected: return_policy
Predicted: return_policy
PASS
Knowledgebase
Question: Does this response adhere toour guidelines
Score:Politeness: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, Coherence: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, Relevancy: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
PASS
Sure - because we’rewithin <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days of thepurchase, I canprocess the return
Question: I want to return a T-shirt Ibought on Amazon on March 3rd.
Ground truth: Eligible for return
PASS

This diagram illustrates a framework for processing a return request using a language model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> system. Here's a 
breakdown of the process:
. **User Input**: The user wants to return a T-shirt purchased on Amazon on March 3rd.
. **Router**: The initial input is processed by a router LLM, which determines the nature of the request. The 
expected and predicted outcomes are both <span style="color: #008000; text-decoration-color: #008000">"return,"</span> and the process passes this evaluation.
. **Return Assistant**: The request is then handled by a return assistant LLM. It interacts with a knowledge base 
to verify the return policy.
. **Knowledge Base**: The system checks the return policy, confirming that the item is eligible for return within 
<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span> days of purchase. The expected and predicted outcomes are <span style="color: #008000; text-decoration-color: #008000">"return_policy,"</span> and this step also passes.
. **Response to User**: The system responds to the user, confirming that the return can be processed because it is 
within the <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">14</span>-day window.
. **Evaluation**: The response is evaluated for adherence to guidelines, scoring <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> for politeness, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for coherence,
and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for relevancy, resulting in a pass.
The framework uses both component evaluations <span style="font-weight: bold">(</span>red dashed lines<span style="font-weight: bold">)</span> and subjective evaluations <span style="font-weight: bold">(</span>orange dashed lines<span style="font-weight: bold">)</span> 
to ensure the process is accurate and user-friendly.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Best practices
Log everything
●
Evals need test cases - log everything as you develop so you can mine your logs for good eval cases
Create a feedback loop
●
●
Build evals into your application so you can quickly run them, iterate and rerun to see the impact
Evals also provide a useful structure for few-shot or ﬁne-tuning examples when optimizing
Employ expert labellers who know the process
● Use experts to help create your eval cases - these need to be as lifelike as possible
Evaluate early and often
●
Evals are something you should build as soon as you have your ﬁrst functioning prompt - you won’t beable to 
optimize without this baseline, so build it early
● Making evals early also forces you to engage with what a good response looks like
. **Log Everything**
   - It's important to log all test cases during development. This allows you to mine your logs for effective 
evaluation cases.
. **Create a Feedback Loop**
   - Integrate evaluations into your application to quickly run, iterate, and rerun them to observe impacts.
   - Evaluations provide a useful structure for few-shot or fine-tuning examples during optimization.
. **Employ Expert Labelers Who Know the Process**
   - Use experts to help create evaluation cases, ensuring they are as realistic as possible.
. **Evaluate Early and Often**
   - Build evaluations as soon as you have a functioning prompt. This baseline is crucial for optimization.
   - Early evaluations help you understand what a good response looks like, facilitating better engagement.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"></pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">## Overview
Evaluation is the process of validating and testing the outputs that your Large Language Model <span style="font-weight: bold">(</span>LLM<span style="font-weight: bold">)</span> applications 
are producing. Strong evaluations, referred to as <span style="color: #008000; text-decoration-color: #008000">"evals,"</span> contribute to creating a more stable and reliable 
application that can withstand changes in code and model updates.
### Example Use Cases
- **Quantify a solution’s reliability**: Measure how dependable your application is.
- **Monitor application performance in production**: Keep track of how well your application performs in real-world
scenarios.
- **Test for regressions**: Ensure that new updates do not negatively impact existing functionality.
### What We’ll Cover
- **What are evals**: Understanding the concept and importance of evaluations.
- **Technical patterns**: Exploring common methods and strategies used in evaluations.
- **Example framework**: Providing a structured approach to implementing evaluations.
- **Best practices**: Sharing tips and guidelines for effective evaluations.
- **Resources**: Offering additional materials for further learning and exploration.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns**
 three types of evaluation methods used in technical assessments:
. **Metric-based Evaluations**:
   - These evaluations use comparison metrics such as BLEU and ROUGE.   - They provide a score that helps in 
filtering and ranking results, making it easier to assess the quality of outputs quantitatively.
. **Component Evaluations**:
   - This method involves comparing the ground truth to predictions.
   - It results in a simple Pass/Fail outcome, which is useful for determining whether specific components meet the
required standards.
. **Subjective Evaluations**:
   - These evaluations rely on a scorecard to assess outputs subjectively.
   - The scorecard can also include a Pass/Fail option, allowing for a more nuanced evaluation that considers 
qualitative aspects.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Metric-based Evaluations
ROUGE is a common metric for evaluating machine summarizations of text. It is specifically used to assess the 
quality of summaries by comparing them to reference summaries.  an example of how ROUGE is applied:
- **Original Text**: This is a detailed description of OpenAI's mission, emphasizing the development of artificial 
general intelligence <span style="font-weight: bold">(</span>AGI<span style="font-weight: bold">)</span> that benefits humanity. It highlights the importance of safety, broad distribution of 
benefits, and avoiding harmful uses or power concentration.
- **Machine Summary**: This is a condensed version of the original text. It focuses on ensuring AGI is safe and 
accessible, avoiding harm and power concentration, and promoting research and collaboration in AI.
- **ROUGE Score**: The score given is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.51162</span>, which quantifies the similarity between the machine-generated 
summary and the original text. A higher score indicates a closer match to the reference summary.
Overall, ROUGE helps in evaluating how well a machine-generated summary captures the essence of the original text.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"># Technical Patterns: Metric-based Evaluations
 the BLEU score, a standard metric used to evaluate machine translation tasks. BLEU stands for Bilingual Evaluation
Understudy and is a method for assessing the quality of text that has been machine-translated from one language to 
another.
### Key Elements:
- **BLEU**: This is a metric specifically designed for evaluating translation tasks. It compares the 
machine-generated translation to one or more reference translations.
- **Original Text**: The example given is in Welsh: <span style="color: #008000; text-decoration-color: #008000">"Y gwir oedd doedden nhw ddim yn dweud celwyddau wedi'r cwbl."</span>
- **Reference Translation**: This is the human-generated translation used as a standard for comparison: <span style="color: #008000; text-decoration-color: #008000">"The truth </span>
<span style="color: #008000; text-decoration-color: #008000">was they were not telling lies after all."</span>
- **Predicted Translation**: This is the translation produced by the machine: <span style="color: #008000; text-decoration-color: #008000">"The truth was they weren't telling </span>
<span style="color: #008000; text-decoration-color: #008000">lies after all."</span>
- **BLEU Score**: The score for this translation is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.39938</span>. This score indicates how closely the machine 
translation matches the reference translation, with a higher score representing a closer match.
The BLEU score is widely used in the field of natural language processing to provide a quantitative measure of 
translation quality.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Technical Patterns: Metric-based Evaluations
**What they’re good for:**
- **Starting Point**: They provide a good starting point for evaluating a new solution, helping to establish 
initial benchmarks.
- **Automated Testing**: These evaluations serve as a useful yardstick for automated testing, particularly in 
determining if a change has caused a significant performance shift.
- **Cost-Effective**: They are cheap and fast, making them accessible for quick assessments.
**What to be aware of:**
- **Context Specificity**: These evaluations are not tailored to specific contexts, which can limit their 
effectiveness in certain situations.
- **Sophistication Needs**: Most customers require more sophisticated evaluations before moving to production, 
indicating that metric-based evaluations might not be sufficient on their own for final decision-making.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Component Evaluations**
Component evaluations, also known as <span style="color: #008000; text-decoration-color: #008000">"unit tests,"</span> focus on assessing a single input/output of an application. The 
goal is to verify that each component functions correctly in isolation by comparing the input to a predefined ideal
result, known as the ground truth.
**Process Overview:**
. **Input Question:**   - The process begins with a question: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>
. **Agent's Role:**
   - The agent receives the question and processes it. The agent's thought process is: <span style="color: #008000; text-decoration-color: #008000">"I don’t know. I should use </span>
<span style="color: #008000; text-decoration-color: #008000">a tool."</span>
   - The agent decides on an action: <span style="color: #008000; text-decoration-color: #008000">"Search."</span>
   - The action input is the original question: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>
. **Search Component:**
   - The search component is tasked with finding the answer. It retrieves the information: <span style="color: #008000; text-decoration-color: #008000">"The current population </span>
<span style="color: #008000; text-decoration-color: #008000">of Canada is 39,566,248 as of Tuesday, May 23, 2023…."</span>
. **Evaluation Steps:**
   - **Correct Action Check:** Is the agent's decision to search the correct action?
   - **Exact Match Comparison:** Does the retrieved answer match the expected result exactly?
   - **Contextual Relevance:** Does the answer use the context provided in the question?
   - **Number Extraction and Comparison:** Extract numbers from both the expected and retrieved answers and compare
them for accuracy.
. **Final Output:**
   - The final output is the verified answer: <span style="color: #008000; text-decoration-color: #008000">"There are 39,566,248 people in Canada as of 2023."</span>
This process ensures that each component of the application is functioning correctly and producing accurate results
by systematically evaluating each step against the ground truth.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Technical Patterns: Subjective Evaluations**
Building an effective scorecard for automated testing is enhanced by incorporating detailed human reviews. This 
process helps identify what is truly valuable. The approach of <span style="color: #008000; text-decoration-color: #008000">"show rather than tell"</span> is recommended for GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, 
meaning that examples of scores like <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>, and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> out of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> should be provided to help the model understand the 
range.
**Example Scorecard:**
- **Role**: You are an evaluation assistant assessing how well the Assistant has answered a customer's query.
 - **Metrics for Assessment**:
  - **Relevance**: Rate the relevance of the search content to the question on a scale from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, where <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> is 
highly relevant and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> is not relevant at all.
  - **Credibility**: Rate the credibility of the sources from <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, where <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span> is an established newspaper, 
government agency, or large company, and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> is unreferenced.
  - **Result**: Determine if the question is answered correctly based on the search content and the user's 
question. Acceptable values are <span style="color: #008000; text-decoration-color: #008000">"correct"</span> or <span style="color: #008000; text-decoration-color: #008000">"incorrect."</span>
- **Output Format**: Provide the evaluation as a JSON document with fields for relevance, credibility, and result.
**Example Evaluation**:
- **User Query**: <span style="color: #008000; text-decoration-color: #008000">"What is the population of Canada?"</span>
- **Assistant's Response**: <span style="color: #008000; text-decoration-color: #008000">"Canada's population was estimated at 39,858,480 on April 1, 2023, by Statistics </span>
<span style="color: #008000; text-decoration-color: #008000">Canada."</span>
- **Evaluation**: `<span style="font-weight: bold">{</span>relevance: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, credibility: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>, result: correct<span style="font-weight: bold">}</span>`
This structured approach ensures clarity and consistency in evaluating the performance of automated systems.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Example Framework**
This framework outlines a method for evaluating the effectiveness of a system by grouping evaluations into test 
suites called <span style="color: #008000; text-decoration-color: #008000">"runs."</span> These runs are executed in batches, and each run's contents are logged and stored at a 
detailed level, known as <span style="color: #008000; text-decoration-color: #008000">"tracing."</span> This allows for investigation of failures, making adjustments, and rerunning 
evaluations.
The table provides a summary of different runs:
- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>**:  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">28</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">18</span> incorrect with correct search results, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
  - Changes: N/A
- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>**:  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">36</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> incorrect with correct search results, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
  - Changes: Model updated to GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>
- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3</span>**:  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">34</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">12</span> incorrect with correct search results, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> incorrect searches
  - Changes: Added few-shot examples
- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>**:  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">42</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">8</span> incorrect with correct search results
  - Changes: Added metadata to search, Prompt engineering for Answer step
- **Run ID <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">5</span>**:  - Model: gpt-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span>-turbo
  - Score: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">48</span>/<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>
  - Annotation Feedback: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> incorrect with correct search results
  - Changes: Prompt engineering to Answer step
This framework emphasizes the importance of detailed logging and iterative improvements to enhance system 
performance.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Overview
Fine-tuning involves adjusting theparameters of pre-trained models on aspeciﬁc dataset or task. This 
processenhances the model's ability to generatemore accurate and relevant responses forthe given context by 
adapting it to thenuances and speciﬁc requirements of thetask at hand.
Example use cases
- Generate output in a consistent
-
format
Process input by following speciﬁcinstructions
What we’ll cover
● When to ﬁne-tune
● Preparing the dataset
● Best practices
● Hyperparameters
● Fine-tuning advances
● Resources

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">What is Fine-tuning
Public Model
Training data
Training
Fine-tunedmodel
Fine-tuning a model consists of training themodel to follow a set of given input/outputexamples.
This will teach the model to behave in acertain way when confronted with a similarinput in the future.
We recommend using <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples
even if the minimum is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>.

Fine-tuning is a process in machine learning where a pre-existing model, known as a public model, is further 
trained using specific training data. This involves adjusting the model to follow a set of given input/output 
examples. The goal is to teach the model to respond in a particular way when it encounters similar inputs in the 
future.
The diagram illustrates this process: starting with a public model, training data is used in a training phase to 
produce a fine-tuned model. This refined model is better suited to specific tasks or datasets.
It is recommended to use <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples for effective fine-tuning, although the minimum requirement is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span> 
examples. This ensures the model learns adequately from the examples provided.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">When to ﬁne-tune
Good for  ✅
Not good for  ❌
●
●
●
●
Following a given format or tone for the
output
Processing the input following speciﬁc,
complex instructions
Improving latency
Reducing token usage
●
●
●
Teaching the model new knowledge
➔ Use RAG or custom models instead
Performing well at multiple, unrelated tasks
➔ Do prompt-engineering or create multiple
FT models instead
Include up-to-date content in responses
➔ Use RAG instead

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Preparing the dataset
Example format
<span style="font-weight: bold">{</span>
<span style="color: #008000; text-decoration-color: #008000">"messages"</span>: <span style="font-weight: bold">[</span>
<span style="font-weight: bold">{</span>
<span style="color: #008000; text-decoration-color: #008000">"role"</span>: <span style="color: #008000; text-decoration-color: #008000">"system"</span>,
<span style="color: #008000; text-decoration-color: #008000">"content"</span>: <span style="color: #008000; text-decoration-color: #008000">"Marv is a factual chatbotthat is also sarcastic."</span>
<span style="font-weight: bold">}</span>,
<span style="font-weight: bold">{</span>
<span style="color: #008000; text-decoration-color: #008000">"role"</span>: <span style="color: #008000; text-decoration-color: #008000">"user"</span>,
<span style="color: #008000; text-decoration-color: #008000">"content"</span>: <span style="color: #008000; text-decoration-color: #008000">"What's the capital ofFrance?"</span>
<span style="font-weight: bold">}</span>,
<span style="font-weight: bold">{</span>
<span style="color: #008000; text-decoration-color: #008000">"role"</span>: <span style="color: #008000; text-decoration-color: #008000">"assistant"</span>,
<span style="color: #008000; text-decoration-color: #008000">"content"</span>: <span style="color: #008000; text-decoration-color: #008000">"Paris, as if everyonedoesn't know that already."</span>
<span style="font-weight: bold">}</span>
<span style="font-weight: bold">]</span>
<span style="font-weight: bold">}</span>
.jsonl
➔ Take the set of instructions and prompts that you
found worked best for the model prior to ﬁne-tuning.Include them in every training example
➔ If you would like to shorten the instructions or
prompts, it may take more training examples to arriveat good results
We recommend using <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples
even if the minimum is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>.

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Best practices
Curate examples carefully
Datasets can be diﬃcult to build, startsmall and invest intentionally.Optimize for fewer high-qualitytraining 
examples.
● Consider “prompt baking”, or using a basicprompt to generate your initial examples
● If your conversations are multi-turn, ensure
your examples are representative
● Collect examples to target issues detected
in evaluation
● Consider the balance &#x26; diversity of data
● Make sure your examples contain all the
information needed in the response
Iterate on hyperparameters
Establish a baseline
Start with the defaults and adjustbased on performance.
● If the model does not appear to converge,
increase the learning rate multiplier
● If the model does not follow the trainingdata as much as expected increase thenumber of epochs
● If the model becomes less diverse than
expected decrease the # of epochs by <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span>
Automate your feedbackpipeline
Introduce automated evaluations tohighlight potential problem cases toclean up and use as training data.
Consider the G-Eval approach ofusing GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> to perform automatedtesting using a scorecard.
Often users start with azero-shot or few-shot prompt tobuild a baseline evaluationbefore graduating to ﬁne-tuning.
Often users start with azero-shot or few-shot prompt tobuild a baseline evaluationOptimize for latency andbefore 
graduating to ﬁne-tuning.
token eﬃciency
When using GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span>, once youhave a baseline evaluation andtraining examples considerﬁne-tuning <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> to get 
similarperformance for less cost andlatency.
Experiment with reducing orremoving system instructionswith subsequent ﬁne-tunedmodel versions.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">Hyperparameters
Epochs
Refers to <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span> full cycle through the training dataset
If you have hundreds of thousands of examples, we would recommendexperimenting with two epochs <span style="font-weight: bold">(</span>or one<span style="font-weight: bold">)</span> to avoid 
overﬁtting.
default: auto <span style="font-weight: bold">(</span>standard is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span><span style="font-weight: bold">)</span>
Batch size
Number of training examples used to train a singleforward &#x26; backward pass
In general, we've found that larger batch sizes tend to work better for larger datasets
default: ~<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>% x N* <span style="font-weight: bold">(</span>max <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">256</span><span style="font-weight: bold">)</span>
*N = number of training examples
Learning rate multiplier
Scaling factor for the original learning rate
We recommend experimenting with values between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.02</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>. We've found thatlarger learning rates often perform better
with larger batch sizes.
default: <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.05</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1</span> or <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>*
*depends on ﬁnal batch size

**Epochs**
- An epoch refers to one complete cycle through the training dataset.
- For datasets with hundreds of thousands of examples, it is recommended to use fewer epochs <span style="font-weight: bold">(</span>one or two<span style="font-weight: bold">)</span> to 
prevent overfitting.
- Default setting is auto, with a standard of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> epochs.
**Batch Size**
- This is the number of training examples used to train in a single forward and backward pass.
- Larger batch sizes are generally more effective for larger datasets.
- The default batch size is approximately <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>% of the total number of training examples <span style="font-weight: bold">(</span>N<span style="font-weight: bold">)</span>, with a maximum of <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">256</span>.
**Learning Rate Multiplier**
- This is a scaling factor for the original learning rate.
- Experimentation with values between <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.02</span> and <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span> is recommended.
- Larger learning rates often yield better results with larger batch sizes.
- Default values are <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.05</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.1</span>, or <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0.2</span>, depending on the final batch size.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"></pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Overview**
Fine-tuning involves adjusting the parameters of pre-trained models on a specific dataset or task. This process 
enhances the model's ability to generate more accurate and relevant responses for the given context by adapting it 
to the nuances and specific requirements of the task at hand.
**Example Use Cases:**
- Generate output in a consistent format.
- Process input by following specific instructions.
**What We’ll Cover:**
- When to fine-tune
- Preparing the dataset
- Best practices
- Hyperparameters
- Fine-tuning advances
- Resources
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">When to Fine-Tune
**Good for:**
- **Following a given format or tone for the output:** Fine-tuning is effective when you need the model to adhere 
to a specific style or structure in its responses.
 - **Processing the input following specific, complex instructions:** It helps in handling detailed and intricate 
instructions accurately.
- **Improving latency:** Fine-tuning can enhance the speed of the model's responses.
- **Reducing token usage:** It can optimize the model to use fewer tokens, making it more efficient.
**Not good for:**
- **Teaching the model new knowledge:** Fine-tuning is not suitable for adding new information to the model. 
Instead, use Retrieval-Augmented Generation <span style="font-weight: bold">(</span>RAG<span style="font-weight: bold">)</span> or custom models.
- **Performing well at multiple, unrelated tasks:** For diverse tasks, it's better to use prompt engineering or 
create multiple fine-tuned models.
- **Including up-to-date content in responses:** Fine-tuning is not ideal for ensuring the model has the latest 
information. RAG is recommended for this purpose.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Preparing the Dataset**
 guidance on preparing a dataset for training a chatbot model. It includes an example format using JSONL <span style="font-weight: bold">(</span>JSON 
Lines<span style="font-weight: bold">)</span> to structure the data. The example shows a conversation with three roles:
. **System**: Sets the context by describing the chatbot as <span style="color: #008000; text-decoration-color: #008000">"Marv is a factual chatbot that is also sarcastic."</span>
. **User**: Asks a question, <span style="color: #008000; text-decoration-color: #008000">"What's the capital of France?"</span>
. **Assistant**: Responds with a sarcastic answer, <span style="color: #008000; text-decoration-color: #008000">"Paris, as if everyone doesn't know that already."</span>
Key recommendations for dataset preparation include:
- Use a set of instructions and prompts that have proven effective for the model before fine-tuning. These should 
be included in every training example.
- If you choose to shorten instructions or prompts, be aware that more training examples may be needed to achieve 
good results.
- It is recommended to use <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">50</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">100</span> examples, even though the minimum required is <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">10</span>.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">**Best Practices**
. **Curate Examples Carefully**
   - Building datasets can be challenging, so start small and focus on high-quality examples.
   - Use <span style="color: #008000; text-decoration-color: #008000">"prompt baking"</span> to generate initial examples.
   - Ensure multi-turn conversations are well-represented.
   - Collect examples to address issues found during evaluation.
   - Balance and diversify your data.
   - Ensure examples contain all necessary information for responses.
. **Iterate on Hyperparameters**
   - Begin with default settings and adjust based on performance.
   - Increase the learning rate multiplier if the model doesn't converge.
   - Increase the number of epochs if the model doesn't follow training data closely.
   - Decrease the number of epochs by <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">2</span> if the model becomes less diverse.
. **Establish a Baseline**
   - Start with zero-shot or few-shot prompts to create a baseline before fine-tuning.
. **Automate Your Feedback Pipeline**
   - Use automated evaluations to identify and clean up problem cases for training data.
   - Consider using the G-Eval approach with GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">4</span> for automated testing with a scorecard.
. **Optimize for Latency and Token Efficiency**
   - After establishing a baseline, consider fine-tuning with GPT-<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">3.5</span> for similar performance at lower cost and 
latency.
   - Experiment with reducing or removing system instructions in subsequent fine-tuned versions.
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">
-------------------------------


</pre>

```python
# Creating the embeddings
# We'll save to a csv file here for testing purposes but this is where you should load content in your vectorDB.
df = pd.DataFrame(clean_content, columns=['content'])
print(df.shape)
df.head()
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="font-weight: bold">(</span><span style="color: #008080; text-decoration-color: #008080; font-weight: bold">88</span>, <span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span><span style="font-weight: bold">)</span>
</pre>

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>content</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Overview\nRetrieval-Augmented Generationenhanc...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>What is RAG\nRetrieve information to Augment t...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>When to use RAG\nGood for  ✅\nNot good for  ❌\...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Technical patterns\nData preparation\nInput pr...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Technical patterns\nData preparation\nchunk do...</td>
    </tr>
  </tbody>
</table>
</div>

```python
embeddings_model = "text-embedding-3-large"

def get_embeddings(text):
    embeddings = client.embeddings.create(
      model="text-embedding-3-small",
      input=text,
      encoding_format="float"
    )
    return embeddings.data[0].embedding
```

```python
df['embeddings'] = df['content'].apply(lambda x: get_embeddings(x))
df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>content</th>
      <th>embeddings</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Overview\nRetrieval-Augmented Generationenhanc...</td>
      <td>[-0.013741373, 0.029359376, 0.054372873, 0.022...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>What is RAG\nRetrieve information to Augment t...</td>
      <td>[-0.018389475, 0.030965596, 0.0056745913, 0.01...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>When to use RAG\nGood for  ✅\nNot good for  ❌\...</td>
      <td>[-0.008419483, 0.021529013, -0.0060885856, 0.0...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Technical patterns\nData preparation\nInput pr...</td>
      <td>[-0.0034501953, 0.03871357, 0.07771268, 0.0041...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Technical patterns\nData preparation\nchunk do...</td>
      <td>[-0.0024594103, 0.023041151, 0.053115055, -0.0...</td>
    </tr>
  </tbody>
</table>
</div>

```python
# Saving locally for later
data_path = "data/parsed_pdf_docs_with_embeddings.csv"
df.to_csv(data_path, index=False)
```

```python
# Optional: load data from saved file
df = pd.read_csv(data_path)
df["embeddings"] = df.embeddings.apply(literal_eval).apply(np.array)
```

## Retrieval-augmented generation

The last step of the process is to generate outputs in response to input queries, after retrieving content as context to reply.

```python
system_prompt = '''
    You will be provided with an input prompt and content as context that can be used to reply to the prompt.
    
    You will do 2 things:
    
    1. First, you will internally assess whether the content provided is relevant to reply to the input prompt. 
    
    2a. If that is the case, answer directly using this content. If the content is relevant, use elements found in the content to craft a reply to the input prompt.

    2b. If the content is not relevant, use your own knowledge to reply or say that you don't know how to respond if your knowledge is not sufficient to answer.
    
    Stay concise with your answer, replying specifically to the input prompt without mentioning additional information provided in the context content.
'''

model="gpt-4o"

def search_content(df, input_text, top_k):
    embedded_value = get_embeddings(input_text)
    df["similarity"] = df.embeddings.apply(lambda x: cosine_similarity(np.array(x).reshape(1,-1), np.array(embedded_value).reshape(1, -1)))
    res = df.sort_values('similarity', ascending=False).head(top_k)
    return res

def get_similarity(row):
    similarity_score = row['similarity']
    if isinstance(similarity_score, np.ndarray):
        similarity_score = similarity_score[0][0]
    return similarity_score

def generate_output(input_prompt, similar_content, threshold = 0.5):
    
    content = similar_content.iloc[0]['content']
    
    # Adding more matching content if the similarity is above threshold
    if len(similar_content) > 1:
        for i, row in similar_content.iterrows():
            similarity_score = get_similarity(row)
            if similarity_score > threshold:
                content += f"\n\n{row['content']}"
            
    prompt = f"INPUT PROMPT:\n{input_prompt}\n-------\nCONTENT:\n{content}"
    
    completion = client.chat.completions.create(
        model=model,
        temperature=0.5,
        messages=[
            {
                "role": "system",
                "content": system_prompt
            },
            {
                "role": "user",
                "content": prompt
            }
        ]
    )

    return completion.choices[0].message.content
```

```python
# Example user queries related to the content
example_inputs = [
    'What are the main models you offer?',
    'Do you have a speech recognition model?',
    'Which embedding model should I use for non-English use cases?',
    'Can I introduce new knowledge in my LLM app using RAG?',
    'How many examples do I need to fine-tune a model?',
    'Which metric can I use to evaluate a summarization task?',
    'Give me a detailed example for an evaluation process where we are looking for a clear answer to compare to a ground truth.',
]
```

```python
# Running the RAG pipeline on each example
for ex in example_inputs:
    print(f"[deep_pink4][bold]QUERY:[/bold] {ex}[/deep_pink4]\n\n")
    matching_content = search_content(df, ex, 3)
    print(f"[grey37][b]Matching content:[/b][/grey37]\n")
    for i, match in matching_content.iterrows():
        print(f"[grey37][i]Similarity: {get_similarity(match):.2f}[/i][/grey37]")
        print(f"[grey37]{match['content'][:100]}{'...' if len(match['content']) > 100 else ''}[/[grey37]]\n\n")
    reply = generate_output(ex, matching_content)
    print(f"[turquoise4][b]REPLY:[/b][/turquoise4]\n\n[spring_green4]{reply}[/spring_green4]\n\n--------------\n\n")
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> What are the main models you offer?</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.42</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">LATEST MODELS</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">This document outlines the latest models available for different endpoints in the Open...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.39</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">26</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">02</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">2024</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">, </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">17:58</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Models - OpenAI API</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">The Moderation models are designed to check whether content co...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.38</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">26</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">02</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">2024</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">, </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">17:58</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Models - OpenAI API</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">MODEL</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">DE S CRIPTION</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">tts-</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">1</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">New  Text-to-speech </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">1</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">The latest tex...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">We offer the following main models:</span>

<span style="color: #00875f; text-decoration-color: #00875f">- **/v1/completions </span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">(</span><span style="color: #00875f; text-decoration-color: #00875f">Legacy</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">)</span><span style="color: #00875f; text-decoration-color: #00875f">**: `gpt-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">3.5</span><span style="color: #00875f; text-decoration-color: #00875f">-turbo-instruct`, `babbage-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">002</span><span style="color: #00875f; text-decoration-color: #00875f">`, `davinci-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">002</span><span style="color: #00875f; text-decoration-color: #00875f">`</span>
<span style="color: #00875f; text-decoration-color: #00875f">- **/v1/embeddings**: `text-embedding-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">3</span><span style="color: #00875f; text-decoration-color: #00875f">-small`, `text-embedding-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">3</span><span style="color: #00875f; text-decoration-color: #00875f">-large`, `text-embedding-ada-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">002</span><span style="color: #00875f; text-decoration-color: #00875f">`</span>
<span style="color: #00875f; text-decoration-color: #00875f">- **/v1/fine_tuning/jobs**: `gpt-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">3.5</span><span style="color: #00875f; text-decoration-color: #00875f">-turbo`, `babbage-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">002</span><span style="color: #00875f; text-decoration-color: #00875f">`, `davinci-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">002</span><span style="color: #00875f; text-decoration-color: #00875f">`</span>
<span style="color: #00875f; text-decoration-color: #00875f">- **/v1/moderations**: `text-moderation-stable`</span>

<span style="color: #00875f; text-decoration-color: #00875f">Additionally, there are enhanced versions like `gpt-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">3.5</span><span style="color: #00875f; text-decoration-color: #00875f">-turbo-16k` and other fine-tuned models.</span>

--------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> Do you have a speech recognition model?</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.51</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**Models - OpenAI API**</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**Text-to-Speech Models:**</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">. **tts-</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">1</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**: This is a new text-to-speech model o...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.50</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">26</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">02</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">2024</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">, </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">17:58</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Models - OpenAI API</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">MODEL</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">DE S CRIPTION</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">tts-</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">1</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">New  Text-to-speech </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">1</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">The latest tex...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.44</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">26</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">02</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">2024</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">, </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">17:58</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Models - OpenAI API</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">ENDP OINT</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">DATA USED</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">FOR TRAINING</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">DEFAULT</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">RETENTION</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">ELIGIBLE FO...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">Yes, there is a speech recognition model called Whisper, which is capable of handling diverse audio inputs and </span>
<span style="color: #00875f; text-decoration-color: #00875f">supports multilingual speech recognition, speech translation, and language identification. The Whisper v2-large </span>
<span style="color: #00875f; text-decoration-color: #00875f">model is accessible via the API under the name </span><span style="color: #00875f; text-decoration-color: #00875f">"whisper-1."</span>

--------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> Which embedding model should I use for non-English use cases?</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.49</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f"># Technical Patterns: Data Preparation - Embeddings</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">## What to Embed?</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">When preparing data for embedd...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.48</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Technical patterns</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Data preparation: embeddings</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">What to embed?</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Depending on your use caseyou might n...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.48</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**Models - OpenAI API**</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**Text-to-Speech Models:**</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">. **tts-</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">1</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**: This is a new text-to-speech model o...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">The content provided does not address which embedding model to use for non-English use cases. For non-English use </span>
<span style="color: #00875f; text-decoration-color: #00875f">cases, you might consider using multilingual models like Google's mBERT or Facebook's XLM-R, which are designed to </span>
<span style="color: #00875f; text-decoration-color: #00875f">handle multiple languages effectively.</span>

--------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> Can I introduce new knowledge in my LLM app using RAG?</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.54</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">What is RAG</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Retrieve information to Augment the model’s knowledge and Generate the output</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">“What is y...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.50</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**Overview**</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Retrieval-Augmented Generation </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">(</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">RAG</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">)</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f"> enhances language models by integrating them with ...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.49</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">When to use RAG</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Good for  ✅</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Not good for  ❌</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Introducing new information to the model</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Teaching ...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">Yes, you can introduce new knowledge in your LLM app using Retrieval-Augmented Generation </span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">(</span><span style="color: #00875f; text-decoration-color: #00875f">RAG</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">)</span><span style="color: #00875f; text-decoration-color: #00875f">. This method allows</span>
<span style="color: #00875f; text-decoration-color: #00875f">the language model to access external knowledge sources, enhancing its responses with up-to-date and contextually </span>
<span style="color: #00875f; text-decoration-color: #00875f">relevant information.</span>

--------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> How many examples do I need to fine-tune a model?</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.71</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">What is Fine-tuning</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Public Model</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Training data</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Training</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Fine-tunedmodel</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Fine-tuning a model consists...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.62</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">When to Fine-Tune</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">**Good for:**</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">- **Following a given format or tone for the output:** Fine-tuning i...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.60</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Best practices</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Curate examples carefully</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Datasets can be diﬃcult to build, startsmall and invest int...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">For effective fine-tuning of a model, it is recommended to use </span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">50</span><span style="color: #00875f; text-decoration-color: #00875f">-</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">100</span><span style="color: #00875f; text-decoration-color: #00875f"> examples. However, the minimum requirement is</span>
<span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">10</span><span style="color: #00875f; text-decoration-color: #00875f"> examples.</span>

--------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> Which metric can I use to evaluate a summarization task?</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.61</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Technical Patterns: Metric-based Evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">ROUGE is a common metric for evaluating machine summari...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.54</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Technical patterns</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Metric-based evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">ROUGE is a common metric for evaluating machine summariz...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.48</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Technical patterns</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Metric-based evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Component evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Subjective evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Compari...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">You can use the ROUGE metric to evaluate a summarization task. ROUGE assesses the quality of summaries by comparing</span>
<span style="color: #00875f; text-decoration-color: #00875f">them to reference summaries, quantifying how well a machine-generated summary captures the essence of the original </span>
<span style="color: #00875f; text-decoration-color: #00875f">text.</span>

--------------


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #af005f; text-decoration-color: #af005f; font-weight: bold">QUERY:</span><span style="color: #af005f; text-decoration-color: #af005f"> Give me a detailed example for an evaluation process where we are looking for a clear answer to compare to a</span>
<span style="color: #af005f; text-decoration-color: #af005f">ground truth.</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">Matching content:</span>

</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.56</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">What are evals</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Example</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Our ground truth matches the predicted answer, so the evaluation passes!</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Eval...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.55</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">What are evals</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Example</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">An evaluation contains a question and a correct answer. We call this the grou...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-style: italic">Similarity: </span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold; font-style: italic">0.55</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Technical patterns</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Metric-based evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Component evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Subjective evaluations</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">●</span>
<span style="color: #5f5f5f; text-decoration-color: #5f5f5f">Compari...</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">[</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f">/</span><span style="color: #5f5f5f; text-decoration-color: #5f5f5f; font-weight: bold">]</span>


</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace"><span style="color: #008787; text-decoration-color: #008787; font-weight: bold">REPLY:</span>

<span style="color: #00875f; text-decoration-color: #00875f">An example of an evaluation process where we look for a clear answer to compare to a ground truth is when </span>
<span style="color: #00875f; text-decoration-color: #00875f">determining the population of a country. In this case, the question is </span><span style="color: #00875f; text-decoration-color: #00875f">"What is the population of Canada?"</span><span style="color: #00875f; text-decoration-color: #00875f"> The </span>
<span style="color: #00875f; text-decoration-color: #00875f">ground truth, or correct answer, is that the population of Canada in </span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">2023</span><span style="color: #00875f; text-decoration-color: #00875f"> is </span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">39</span><span style="color: #00875f; text-decoration-color: #00875f">,</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">566</span><span style="color: #00875f; text-decoration-color: #00875f">,</span><span style="color: #00875f; text-decoration-color: #00875f; font-weight: bold">248</span><span style="color: #00875f; text-decoration-color: #00875f"> people. The predicted </span>
<span style="color: #00875f; text-decoration-color: #00875f">answer, obtained through a search tool, is </span><span style="color: #00875f; text-decoration-color: #00875f">"There are 39,566,248 people in Canada as of 2023."</span><span style="color: #00875f; text-decoration-color: #00875f"> Since the predicted </span>
<span style="color: #00875f; text-decoration-color: #00875f">answer matches the ground truth, the evaluation is successful. This process is used to verify the accuracy of </span>
<span style="color: #00875f; text-decoration-color: #00875f">information provided by a language model or other predictive tools.</span>

--------------


</pre>

## Wrapping up

In this notebook, we have learned how to develop a basic RAG pipeline based on PDF documents. This includes:

- How to parse pdf documents, taking slide decks and an export from an HTML page as examples, using a python library as well as GPT-4o to interpret the visuals
- How to process the extracted content, clean it and chunk it into several pieces
- How to embed the processed content using OpenAI embeddings
- How to retrieve content that is relevant to an input query
- How to use GPT-4o to generate an answer using the retrieved content as context

If you want to explore further, consider these optimisations:

- Playing around with the prompts provided as examples
- Chunking the content further and adding metadata as context to each chunk
- Adding rule-based filtering on the retrieval results or re-ranking results to surface to most relevant content

You can apply the techniques covered in this notebook to multiple use cases, such as assistants that can access your proprietary data, customer service or FAQ bots that can read from your internal policies, or anything that requires leveraging rich documents that would be better understood as images.