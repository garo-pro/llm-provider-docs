# Multiclass Classification for Transactions

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

For this notebook we will be looking to classify a public dataset of transactions into a number of categories that we have predefined. These approaches should be replicable to any multiclass classification use case where we are trying to fit transactional data into predefined categories, and by the end of running through this you should have a few approaches for dealing with both labelled and unlabelled datasets.

The different approaches we'll be taking in this notebook are:
- **Zero-shot Classification:** First we'll do zero shot classification to put transactions in one of five named buckets using only a prompt for guidance
- **Classification with Embeddings:** Following this we'll create embeddings on a labelled dataset, and then use a traditional classification model to test their effectiveness at identifying our categories
- **Fine-tuned Classification:** Lastly we'll produce a fine-tuned model trained on our labelled dataset to see how this compares to the zero-shot and few-shot classification approaches

## Setup

```python
%load_ext autoreload
%autoreload
%pip install openai 'openai[datalib]' 'openai[embeddings]' transformers scikit-learn matplotlib plotly pandas scipy
```

```python
import openai
import pandas as pd
import numpy as np
import json
import os

COMPLETIONS_MODEL = "gpt-4"
os.environ["OPENAI_API_KEY"] = "<your-api-key>"
client = openai.OpenAI()
```

### Load dataset

We're using a public transaction dataset of transactions over £25k for the Library of Scotland. The dataset has three features that we'll be using:
- Supplier: The name of the supplier
- Description: A text description of the transaction
- Value: The value of the transaction in GBP

**Source**:

https://data.nls.uk/data/organisational-data/transactions-over-25k/

```python
transactions = pd.read_csv('./data/25000_spend_dataset_current.csv', encoding= 'unicode_escape')
print(f"Number of transactions: {len(transactions)}")
print(transactions.head())
```

```text
Number of transactions: 359
         Date                      Supplier                 Description  \
0  21/04/2016          M & J Ballantyne Ltd       George IV Bridge Work   
1  26/04/2016                  Private Sale   Literary & Archival Items   
2  30/04/2016     City Of Edinburgh Council         Non Domestic Rates    
3  09/05/2016              Computacenter Uk                 Kelvin Hall   
4  09/05/2016  John Graham Construction Ltd  Causewayside Refurbishment   

   Transaction value (£)  
0                35098.0  
1                30000.0  
2                40800.0  
3                72835.0  
4                64361.0
```

## Zero-shot Classification

We'll first assess the performance of the base models at classifying these transactions using a simple prompt. We'll provide the model with 5 categories and a catch-all of "Could not classify" for ones that it cannot place.

```python
zero_shot_prompt = '''You are a data expert working for the National Library of Scotland.
You are analysing all transactions over £25,000 in value and classifying them into one of five categories.
The five categories are Building Improvement, Literature & Archive, Utility Bills, Professional Services and Software/IT.
If you can't tell what it is, say Could not classify

Transaction:

Supplier: {}
Description: {}
Value: {}

The classification is:'''

def format_prompt(transaction):
    return zero_shot_prompt.format(transaction['Supplier'], transaction['Description'], transaction['Transaction value (£)'])

def classify_transaction(transaction):

    
    prompt = format_prompt(transaction)
    messages = [
        {"role": "system", "content": prompt},
    ]
    completion_response = openai.chat.completions.create(
                            messages=messages,
                            temperature=0,
                            max_tokens=5,
                            top_p=1,
                            frequency_penalty=0,
                            presence_penalty=0,
                            model=COMPLETIONS_MODEL)
    label = completion_response.choices[0].message.content.replace('\n','')
    return label
```

```python
# Get a test transaction
transaction = transactions.iloc[0]
# Use our completion function to return a prediction
print(f"Transaction: {transaction['Supplier']} {transaction['Description']} {transaction['Transaction value (£)']}")
print(f"Classification: {classify_transaction(transaction)}")
```

```text
Transaction: M & J Ballantyne Ltd George IV Bridge Work 35098.0
Classification: Building Improvement
```

Our first attempt is correct, M & J Ballantyne Ltd are a house builder and the work they performed is indeed Building Improvement.

Lets expand the sample size to 25 and see how it performs, again with just a simple prompt to guide it

```python
test_transactions = transactions.iloc[:25]
test_transactions['Classification'] = test_transactions.apply(lambda x: classify_transaction(x),axis=1)
```

```text
/var/folders/3n/79rgh27s6l7_l91b9shw0_nr0000gp/T/ipykernel_81921/2775604370.py:2: SettingWithCopyWarning: 
A value is trying to be set on a copy of a slice from a DataFrame.
Try using .loc[row_indexer,col_indexer] = value instead

See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
  test_transactions['Classification'] = test_transactions.apply(lambda x: classify_transaction(x),axis=1)
```

```python
test_transactions['Classification'].value_counts()
```

```text
Classification
Building Improvement    17
Literature & Archive     3
Software/IT              2
Could not classify       2
Utility Bills            1
Name: count, dtype: int64
```

```python
test_transactions.head(25)
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>Supplier</th>
      <th>Description</th>
      <th>Transaction value (£)</th>
      <th>Classification</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>21/04/2016</td>
      <td>M &#x26; J Ballantyne Ltd</td>
      <td>George IV Bridge Work</td>
      <td>35098.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>1</th>
      <td>26/04/2016</td>
      <td>Private Sale</td>
      <td>Literary &#x26; Archival Items</td>
      <td>30000.0</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>2</th>
      <td>30/04/2016</td>
      <td>City Of Edinburgh Council</td>
      <td>Non Domestic Rates</td>
      <td>40800.0</td>
      <td>Utility Bills</td>
    </tr>
    <tr>
      <th>3</th>
      <td>09/05/2016</td>
      <td>Computacenter Uk</td>
      <td>Kelvin Hall</td>
      <td>72835.0</td>
      <td>Software/IT</td>
    </tr>
    <tr>
      <th>4</th>
      <td>09/05/2016</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>64361.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>5</th>
      <td>09/05/2016</td>
      <td>A McGillivray</td>
      <td>Causewayside Refurbishment</td>
      <td>53690.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>6</th>
      <td>16/05/2016</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>365344.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>7</th>
      <td>23/05/2016</td>
      <td>Computacenter Uk</td>
      <td>Kelvin Hall</td>
      <td>26506.0</td>
      <td>Software/IT</td>
    </tr>
    <tr>
      <th>8</th>
      <td>23/05/2016</td>
      <td>ECG Facilities Service</td>
      <td>Facilities Management Charge</td>
      <td>32777.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>9</th>
      <td>23/05/2016</td>
      <td>ECG Facilities Service</td>
      <td>Facilities Management Charge</td>
      <td>32777.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>10</th>
      <td>30/05/2016</td>
      <td>ALDL</td>
      <td>ALDL Charges</td>
      <td>32317.0</td>
      <td>Could not classify</td>
    </tr>
    <tr>
      <th>11</th>
      <td>10/06/2016</td>
      <td>Wavetek Ltd</td>
      <td>Kelvin Hall</td>
      <td>87589.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>12</th>
      <td>10/06/2016</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>381803.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>13</th>
      <td>28/06/2016</td>
      <td>ECG Facilities Service</td>
      <td>Facilities Management Charge</td>
      <td>32832.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>14</th>
      <td>30/06/2016</td>
      <td>Glasgow City Council</td>
      <td>Kelvin Hall</td>
      <td>1700000.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>15</th>
      <td>11/07/2016</td>
      <td>Wavetek Ltd</td>
      <td>Kelvin Hall</td>
      <td>65692.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>16</th>
      <td>11/07/2016</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>139845.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>17</th>
      <td>15/07/2016</td>
      <td>Sotheby'S</td>
      <td>Literary &#x26; Archival Items</td>
      <td>28500.0</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>18</th>
      <td>18/07/2016</td>
      <td>Christies</td>
      <td>Literary &#x26; Archival Items</td>
      <td>33800.0</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>19</th>
      <td>25/07/2016</td>
      <td>A McGillivray</td>
      <td>Causewayside Refurbishment</td>
      <td>30113.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>20</th>
      <td>31/07/2016</td>
      <td>ALDL</td>
      <td>ALDL Charges</td>
      <td>32317.0</td>
      <td>Could not classify</td>
    </tr>
    <tr>
      <th>21</th>
      <td>08/08/2016</td>
      <td>ECG Facilities Service</td>
      <td>Facilities Management Charge</td>
      <td>32795.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>22</th>
      <td>15/08/2016</td>
      <td>Creative Video Productions Ltd</td>
      <td>Kelvin Hall</td>
      <td>26866.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>23</th>
      <td>15/08/2016</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>196807.0</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>24</th>
      <td>24/08/2016</td>
      <td>ECG Facilities Service</td>
      <td>Facilities Management Charge</td>
      <td>32795.0</td>
      <td>Building Improvement</td>
    </tr>
  </tbody>
</table>
</div>

Initial results are pretty good even with no labelled examples! The ones that it could not classify were tougher cases with few clues as to their topic, but maybe if we clean up the labelled dataset to give more examples we can get better performance.

## Classification with Embeddings

Lets create embeddings from the small set that we've classified so far - we've made a set of labelled examples by running the zero-shot classifier on 101 transactions from our dataset and manually correcting the 15 **Could not classify** results that we got

### Create embeddings

This initial section reuses the approach from the [Get_embeddings_from_dataset Notebook](https://developers.openai.com/cookbook/examples/Get_embeddings_from_dataset.ipynb) to create embeddings from a combined field concatenating all of our features

```python
df = pd.read_csv('./data/labelled_transactions.csv')
df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>Supplier</th>
      <th>Description</th>
      <th>Transaction value (£)</th>
      <th>Classification</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>15/08/2016</td>
      <td>Creative Video Productions Ltd</td>
      <td>Kelvin Hall</td>
      <td>26866</td>
      <td>Other</td>
    </tr>
    <tr>
      <th>1</th>
      <td>29/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>74806</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>2</th>
      <td>29/05/2017</td>
      <td>Morris &#x26; Spottiswood Ltd</td>
      <td>George IV Bridge Work</td>
      <td>56448</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>3</th>
      <td>31/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>164691</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>4</th>
      <td>24/07/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>27926</td>
      <td>Building Improvement</td>
    </tr>
  </tbody>
</table>
</div>

```python
df['combined'] = "Supplier: " + df['Supplier'].str.strip() + "; Description: " + df['Description'].str.strip() + "; Value: " + str(df['Transaction value (£)']).strip()
df.head(2)
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>Supplier</th>
      <th>Description</th>
      <th>Transaction value (£)</th>
      <th>Classification</th>
      <th>combined</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>15/08/2016</td>
      <td>Creative Video Productions Ltd</td>
      <td>Kelvin Hall</td>
      <td>26866</td>
      <td>Other</td>
      <td>Supplier: Creative Video Productions Ltd; Desc...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>29/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>74806</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
    </tr>
  </tbody>
</table>
</div>

```python
from transformers import GPT2TokenizerFast
tokenizer = GPT2TokenizerFast.from_pretrained("gpt2")

df['n_tokens'] = df.combined.apply(lambda x: len(tokenizer.encode(x)))
len(df)
```

```text
101
```

```python
embedding_path = './data/transactions_with_embeddings_100.csv'
```

```python
from utils.embeddings_utils import get_embedding
df['babbage_similarity'] = df.combined.apply(lambda x: get_embedding(x))
df['babbage_search'] = df.combined.apply(lambda x: get_embedding(x))
df.to_csv(embedding_path)
```

### Use embeddings for classification

Now that we have our embeddings, let see if classifying these into the categories we've named gives us any more success.

For this we'll use a template from the [Classification_using_embeddings](https://developers.openai.com/cookbook/examples/Classification_using_embeddings.ipynb) notebook

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
from ast import literal_eval

fs_df = pd.read_csv(embedding_path)
fs_df["babbage_similarity"] = fs_df.babbage_similarity.apply(literal_eval).apply(np.array)
fs_df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Unnamed: 0</th>
      <th>Date</th>
      <th>Supplier</th>
      <th>Description</th>
      <th>Transaction value (£)</th>
      <th>Classification</th>
      <th>combined</th>
      <th>n_tokens</th>
      <th>babbage_similarity</th>
      <th>babbage_search</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>15/08/2016</td>
      <td>Creative Video Productions Ltd</td>
      <td>Kelvin Hall</td>
      <td>26866</td>
      <td>Other</td>
      <td>Supplier: Creative Video Productions Ltd; Desc...</td>
      <td>136</td>
      <td>[-0.02898375503718853, -0.02881557121872902, 0...</td>
      <td>[-0.02879939414560795, -0.02867320366203785, 0...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>29/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>74806</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
      <td>140</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>29/05/2017</td>
      <td>Morris &#x26; Spottiswood Ltd</td>
      <td>George IV Bridge Work</td>
      <td>56448</td>
      <td>Building Improvement</td>
      <td>Supplier: Morris &#x26; Spottiswood Ltd; Descriptio...</td>
      <td>141</td>
      <td>[0.013581369072198868, -0.003978211898356676, ...</td>
      <td>[0.013593776151537895, -0.0037341134157031775,...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>31/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>164691</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
      <td>140</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>24/07/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>27926</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
      <td>140</td>
      <td>[-0.02408558875322342, -0.02881370671093464, 0...</td>
      <td>[-0.024109570309519768, -0.02880912832915783, ...</td>
    </tr>
  </tbody>
</table>
</div>

```python
X_train, X_test, y_train, y_test = train_test_split(
    list(fs_df.babbage_similarity.values), fs_df.Classification, test_size=0.2, random_state=42
)

clf = RandomForestClassifier(n_estimators=100)
clf.fit(X_train, y_train)
preds = clf.predict(X_test)
probas = clf.predict_proba(X_test)

report = classification_report(y_test, preds)
print(report)
```

```text
                      precision    recall  f1-score   support

Building Improvement       0.92      1.00      0.96        11
Literature & Archive       1.00      1.00      1.00         3
               Other       0.00      0.00      0.00         1
         Software/IT       1.00      1.00      1.00         1
       Utility Bills       1.00      1.00      1.00         5

            accuracy                           0.95        21
           macro avg       0.78      0.80      0.79        21
        weighted avg       0.91      0.95      0.93        21
```

```text
/Users/vishnu/code/openai-cookbook/cookbook_env/lib/python3.11/site-packages/sklearn/metrics/_classification.py:1565: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
  _warn_prf(average, modifier, f"{metric.capitalize()} is", len(result))
/Users/vishnu/code/openai-cookbook/cookbook_env/lib/python3.11/site-packages/sklearn/metrics/_classification.py:1565: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
  _warn_prf(average, modifier, f"{metric.capitalize()} is", len(result))
/Users/vishnu/code/openai-cookbook/cookbook_env/lib/python3.11/site-packages/sklearn/metrics/_classification.py:1565: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
  _warn_prf(average, modifier, f"{metric.capitalize()} is", len(result))
```

Performance for this model is pretty strong, so creating embeddings and using even a simpler classifier looks like an effective approach as well, with the zero-shot classifier helping us do the initial classification of the unlabelled dataset.

Lets take it one step further and see if a fine-tuned model trained on this same labelled datasets gives us comparable results

## Fine-tuned Transaction Classification

For this use case we're going to try to improve on the few-shot classification from above by training a fine-tuned model on the same labelled set of 101 transactions and applying this fine-tuned model on group of unseen transactions

### Building Fine-tuned Classifier

We'll need to do some data prep first to get our data ready. This will take the following steps:
- To prepare our training and validation sets, we'll create a set of message sequences. The first message for each will be the user prompt formatted with the details of the transaction, and the final message will be the expected classification response from the model
- Our test set will contain the initial user prompt for each transaction, along with the corresponding expected class label. We will then use the fine-tuned model to generate the actual classification for each transaction.

```python
ft_prep_df = fs_df.copy()
len(ft_prep_df)
```

```text
101
```

```python
ft_prep_df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Unnamed: 0</th>
      <th>Date</th>
      <th>Supplier</th>
      <th>Description</th>
      <th>Transaction value (£)</th>
      <th>Classification</th>
      <th>combined</th>
      <th>n_tokens</th>
      <th>babbage_similarity</th>
      <th>babbage_search</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>15/08/2016</td>
      <td>Creative Video Productions Ltd</td>
      <td>Kelvin Hall</td>
      <td>26866</td>
      <td>Other</td>
      <td>Supplier: Creative Video Productions Ltd; Desc...</td>
      <td>136</td>
      <td>[-0.028885245323181152, -0.028660893440246582,...</td>
      <td>[-0.02879939414560795, -0.02867320366203785, 0...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>29/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>74806</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
      <td>140</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
      <td>[-0.02414606139063835, -0.02883070334792137, 0...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>29/05/2017</td>
      <td>Morris &#x26; Spottiswood Ltd</td>
      <td>George IV Bridge Work</td>
      <td>56448</td>
      <td>Building Improvement</td>
      <td>Supplier: Morris &#x26; Spottiswood Ltd; Descriptio...</td>
      <td>141</td>
      <td>[0.013593776151537895, -0.0037341134157031775,...</td>
      <td>[0.013561442494392395, -0.004199974238872528, ...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>31/05/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>164691</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
      <td>140</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>24/07/2017</td>
      <td>John Graham Construction Ltd</td>
      <td>Causewayside Refurbishment</td>
      <td>27926</td>
      <td>Building Improvement</td>
      <td>Supplier: John Graham Construction Ltd; Descri...</td>
      <td>140</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
      <td>[-0.024112487211823463, -0.02881261520087719, ...</td>
    </tr>
  </tbody>
</table>
</div>

```python
classes = list(set(ft_prep_df['Classification']))
class_df = pd.DataFrame(classes).reset_index()
class_df.columns = ['class_id','class']
class_df  , len(class_df)
```

```text
(   class_id                 class
 0         0                 Other
 1         1  Literature & Archive
 2         2           Software/IT
 3         3         Utility Bills
 4         4  Building Improvement,
 5)
```

```python
ft_df_with_class = ft_prep_df.merge(class_df,left_on='Classification',right_on='class',how='inner')

# Creating a list of messages for the fine-tuning job. The user message is the prompt, and the assistant message is the response from the model
ft_df_with_class['messages'] = ft_df_with_class.apply(lambda x: [{"role": "user", "content": format_prompt(x)}, {"role": "assistant", "content": x['class']}],axis=1)
ft_df_with_class[['messages', 'class']].head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>messages</th>
      <th>class</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Other</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Building Improvement</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Building Improvement</td>
    </tr>
  </tbody>
</table>
</div>

```python
# Create train/validation split
samples = ft_df_with_class["messages"].tolist()
train_df, valid_df = train_test_split(samples, test_size=0.2, random_state=42)

def write_to_jsonl(list_of_messages, filename):
    with open(filename, "w+") as f:
        for messages in list_of_messages:
            object = {  
                "messages": messages
            }
            f.write(json.dumps(object) + "\n")
```

```python
# Write the train/validation split to jsonl files
train_file_name, valid_file_name = "transactions_grouped_train.jsonl", "transactions_grouped_valid.jsonl"
write_to_jsonl(train_df, train_file_name)
write_to_jsonl(valid_df, valid_file_name)
```

```python
# Upload the files to OpenAI
train_file = client.files.create(file=open(train_file_name, "rb"), purpose="fine-tune")
valid_file = client.files.create(file=open(valid_file_name, "rb"), purpose="fine-tune")
```

```python
# Create the fine-tuning job
fine_tuning_job = client.fine_tuning.jobs.create(training_file=train_file.id, validation_file=valid_file.id, model="gpt-4o-2024-08-06")
# Get the fine-tuning job status and model name
status = client.fine_tuning.jobs.retrieve(fine_tuning_job.id)
```

```python
# Once the fine-tuning job is complete, you can retrieve the model name from the job status
fine_tuned_model = client.fine_tuning.jobs.retrieve(fine_tuning_job.id).fine_tuned_model
print(f"Fine tuned model id: {fine_tuned_model}")
```

```text
Fine tuned model id: ft:gpt-4o-2024-08-06:openai::BKr3Xy8U
```

### Applying Fine-tuned Classifier

Now we'll apply our classifier to see how it performs. We only had 31 unique observations in our training set and 8 in our validation set, so lets see how the performance is

```python
# Create a test set with the expected class labels
test_set = pd.read_json(valid_file_name, lines=True)
test_set['expected_class'] = test_set.apply(lambda x: x['messages'][-1]['content'], axis=1)
test_set.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>messages</th>
      <th>expected_class</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Utility Bills</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Building Improvement</td>
    </tr>
  </tbody>
</table>
</div>

```python
# Apply the fine-tuned model to the test set
test_set['response'] = test_set.apply(lambda x: openai.chat.completions.create(model=fine_tuned_model, messages=x['messages'][:-1], temperature=0),axis=1)
test_set['predicted_class'] = test_set.apply(lambda x: x['response'].choices[0].message.content, axis=1)

test_set.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>messages</th>
      <th>expected_class</th>
      <th>response</th>
      <th>predicted_class</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Utility Bills</td>
      <td>ChatCompletion(id='chatcmpl-BKrC0S1wQSfM9ZQfcC...</td>
      <td>Utility Bills</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Literature &#x26; Archive</td>
      <td>ChatCompletion(id='chatcmpl-BKrC1BTr0DagbDkC2s...</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Literature &#x26; Archive</td>
      <td>ChatCompletion(id='chatcmpl-BKrC1H3ZeIW5cz2Owr...</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Literature &#x26; Archive</td>
      <td>ChatCompletion(id='chatcmpl-BKrC1wdhaMP0Q7YmYx...</td>
      <td>Literature &#x26; Archive</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[{'role': 'user', 'content': 'You are a data e...</td>
      <td>Building Improvement</td>
      <td>ChatCompletion(id='chatcmpl-BKrC20c5pkpngy1xDu...</td>
      <td>Building Improvement</td>
    </tr>
  </tbody>
</table>
</div>

```python
# Calculate the accuracy of the predictions
from sklearn.metrics import f1_score
test_set['result'] = test_set.apply(lambda x: str(x['predicted_class']).strip() == str(x['expected_class']).strip(), axis = 1)
test_set['result'].value_counts()

print(test_set['result'].value_counts())

print("F1 Score: ", f1_score(test_set['expected_class'], test_set['predicted_class'], average="weighted"))
print("Raw Accuracy: ", test_set['result'].value_counts()[True] / len(test_set))
```

```text
result
True     20
False     1
Name: count, dtype: int64
F1 Score:  0.9296066252587991
Raw Accuracy:  0.9523809523809523
```