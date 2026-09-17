# How to test and evaluate LLMs for SQL generation

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

LLMs are fundamentally nondeterministic. This attribute makes their responses creative and dynamic. However, this trait poses significant challenges in achieving consistency, a crucial aspect for integrating LLMs into production environments.

The key to harnessing the potential of LLMs in practical applications lies in consistent and systematic evaluation. This enables the identification and rectification of inconsistencies and helps with monitoring progress over time as the application evolves.

## Scope of this notebook

This notebook aims to demonstrate a framework for evaluating LLMs, particularly focusing on:

* **Unit Testing:** Essential for assessing individual components of the application.
* **Evaluation Metrics:** Methods to quantitatively measure the model's effectiveness.
* **Runbook Documentation:** A record of historical evaluations to track progress and regression.

This example focuses on a natural language to SQL use case - code generation use cases fit well with this approach when you combine **code validation** with **code execution**, so your application can test code for real as it is generated to ensure consistency.

Although this notebook uses SQL generation use case to demonstrate the concept, the approach is generic and can be applied to a wide variety of LLM-driven applications.

We will use two versions of a prompt to perform SQL generation.  We will then use the unit tests and evaluation functions to test the performance of the prompts.  Specifically, in this demonstration, we will evaluate:

1. The consistency of JSON response.
2. Syntactic correctness of SQL in response.


## Table of contents

1. **[Setup](#Setup):** Install required libraries, download data consisting of SQL queries and corresponding natural language translations.
2. **[Test Development](#Test-development):** Create unit tests and define evaluation metrics for the SQL generation process.
3. **[Evaluation](#Evaluation):** Conduct tests using different prompts to assess the impact on performance.
4. **[Reporting](#Report):** Compile a report that succinctly presents the performance differences observed across various tests.

## Setup

Import our libraries and the dataset we'll use, which is the natural language to SQL [b-mc2/sql-create-context](https://huggingface.co/datasets/b-mc2/sql-create-context) dataset from HuggingFace.

```python
# Uncomment this to install all necessary dependencies
# !pip install openai datasets pandas pydantic matplotlib python-dotenv numpy tqdm
```

```python
from datasets import load_dataset
from openai import OpenAI
import pandas as pd
import pydantic
import os
import sqlite3
from sqlite3 import Error
from pprint import pprint
import matplotlib.pyplot as plt
import numpy as np
from dotenv import load_dotenv
from tqdm.notebook import tqdm
from IPython.display import HTML, display

# Loads key from local .env file to setup API KEY in env variables
%reload_ext dotenv
%dotenv
    
GPT_MODEL = 'gpt-4o'
dataset = load_dataset("b-mc2/sql-create-context")

print(dataset['train'].num_rows, "rows")
```

```text
78577 rows
```

### Looking at the dataset

We use Huggingface datasets library to download SQL create context dataset.  This dataset consists of:

1. Question, expressed in natural language
2. Answer, expressed in SQL designed to answer the question in natural language.
3. Context, expressed as a CREATE SQL statement, that describes the table that may be used to answer the question.

In our demonstration today, we will use LLM to attempt to answer the question (in natural language).  The LLM will be expected to generate a CREATE SQL statement to create a context suitable to answer the user question and a corresponding SELECT SQL query designed to answer the user question completely.

The dataset looks like this:

```python
sql_df = dataset['train'].to_pandas()
sql_df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>answer</th>
      <th>question</th>
      <th>context</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>SELECT COUNT(*) FROM head WHERE age &gt; 56</td>
      <td>How many heads of the departments are older th...</td>
      <td>CREATE TABLE head (age INTEGER)</td>
    </tr>
    <tr>
      <th>1</th>
      <td>SELECT name, born_state, age FROM head ORDER B...</td>
      <td>List the name, born state and age of the heads...</td>
      <td>CREATE TABLE head (name VARCHAR, born_state VA...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>SELECT creation, name, budget_in_billions FROM...</td>
      <td>List the creation year, name and budget of eac...</td>
      <td>CREATE TABLE department (creation VARCHAR, nam...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>SELECT MAX(budget_in_billions), MIN(budget_in_...</td>
      <td>What are the maximum and minimum budget of the...</td>
      <td>CREATE TABLE department (budget_in_billions IN...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>SELECT AVG(num_employees) FROM department WHER...</td>
      <td>What is the average number of employees of the...</td>
      <td>CREATE TABLE department (num_employees INTEGER...</td>
    </tr>
  </tbody>
</table>
</div>

## Test development

To test the output of the LLM generations, we'll develop two unit tests and an evaluation, which will combine to give us a basic evaluation framework to grade the quality of our LLM iterations.

To re-iterate, our purpose is to measure the correctness and consistency of LLM output given our questions.

### Unit tests

Unit tests should test the most granular components of your LLM application.

For this section we'll develop unit tests to test the following:
- `test_valid_schema` will check that a parseable `create` and `select` statement are returned by the LLM.
- `test_llm_sql` will execute both the `create` and `select` statements on a `sqlite` database to ensure they are syntactically correct.

```python
from pydantic import BaseModel


class LLMResponse(BaseModel):
    """This is the structure that we expect the LLM to respond with.

    The LLM should respond with a JSON string with `create` and `select` fields.
    """
    create: str
    select: str
```

#### Prompting the LLM

For this demonstration purposes, we use a fairly simple prompt requesting GPT to generate a `(context, answer)` pair. `context` is the `CREATE` SQL statement, and `answer` is the `SELECT` SQL statement. We supply the natural language question as part of the prompt.  We request the response to be in JSON format, so that it can be parsed easily.

```python
system_prompt = """Translate this natural language request into a JSON
object containing two SQL queries. The first query should be a CREATE 
tatement for a table answering the user's request, while the second
should be a SELECT query answering their question."""

# Sending the message array to GPT, requesting a response (ensure that you
# have API key loaded to Env for this step)
client = OpenAI()

def get_response(system_prompt, user_message, model=GPT_MODEL):
    messages = []
    messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": user_message})

    response = client.beta.chat.completions.parse(
        model=GPT_MODEL,
        messages=messages,
        response_format=LLMResponse,
    )
    return response.choices[0].message.content

question = sql_df.iloc[0]['question']
content = get_response(system_prompt, question)
print("Question:", question)
print("Answer:", content)
```

```text
Question: How many heads of the departments are older than 56 ?
Answer: {"create":"CREATE TABLE DepartmentHeads (\n    id INT PRIMARY KEY,\n    name VARCHAR(100),\n    age INT,\n    department VARCHAR(100)\n);","select":"SELECT COUNT(*) AS NumberOfHeadsOlderThan56 \nFROM DepartmentHeads \nWHERE age > 56;"}
```

#### Check JSON formatting

Our first simple unit test checks that the LLM response is parseable into the `LLMResponse` Pydantic class that we've defined.

We'll test that our first response passes, then create a failing example to check that the check fails. This logic will be wrapped in a simple function `test_valid_schema`.

We expect GPT to respond with a valid SQL, we can validate this using LLMResponse base model.  `test_valid_schema` is designed to help us validate this.

```python
def test_valid_schema(content):
    """Tests whether the content provided can be parsed into our Pydantic model."""
    try:
        LLMResponse.model_validate_json(content)
        return True
    # Catch pydantic's validation errors:
    except pydantic.ValidationError as exc:
        print(f"ERROR: Invalid schema: {exc}")
        return False
```

```python
test_valid_schema(content)
```

```text
True
```

#### Testing negative scenario

To simulate a scenario in which we get an invalid JSON response from GPT, we hardcode an invalid JSON as response.  We expect `test_valid_schema` function to throw an exception.

```python
failing_query = 'CREATE departments, select * from departments'
test_valid_schema(failing_query)
```

```text
ERROR: Invalid schema: 1 validation error for LLMResponse
  Invalid JSON: expected value at line 1 column 1 [type=json_invalid, input_value='CREATE departments, select * from departments', input_type=str]
    For further information visit https://errors.pydantic.dev/2.10/v/json_invalid
```

```text
False
```

As expected, we get an exception thrown from the `test_valid_schema` function.

### Test SQL queries

Next we'll validate the correctness of the SQL.  This test is designed to validate:

1. The CREATE SQL returned in GPT response is syntactically correct.
2. The SELECT SQL returned in the GPT response is syntactically correct.

To achieve this, we will use a SQLite instance. We will send the returned SQL statements to the SQLite instance. If the statements are valid, SQLite will accept and execute them; otherwise we will expect an exception to be thrown.

The `create_connection` function below will set up a SQLite instance (in-memory by default) and create a connection to be used later.

```python
# Set up SQLite to act as our test database
def create_connection(db_file=":memory:"):
    """create a database connection to a SQLite database"""
    try:
        conn = sqlite3.connect(db_file)
        # print(sqlite3.version)
    except Error as e:
        print(e)
        return None

    return conn

def close_connection(conn):
    """close a database connection"""
    try:
        conn.close()
    except Error as e:
        print(e)


conn = create_connection()
```

Next, we will create the following functions to carry out the syntactical correctness checks.


- `test_create`: Function testing if the CREATE SQL statement succeeds.
- `test_select`: Function testing if the SELECT SQL statement succeeds.
- `test_llm_sql`: Wrapper function executing the two tests above.

```python
def test_select(conn, cursor, select, should_log=True):
    """Tests that a SQLite select query can be executed successfully."""
    try:
        if should_log:
            print(f"Testing select query: {select}")
        cursor.execute(select)
        record = cursor.fetchall()
        if should_log:
            print(f"Result of query: {record}")

        return True

    except sqlite3.Error as error:
        if should_log:
            print("Error while executing select query:", error)
        return False


def test_create(conn, cursor, create, should_log=True):
    """Tests that a SQLite create query can be executed successfully"""
    try:
        if should_log:
            print(f"Testing create query: {create}")
        cursor.execute(create)
        conn.commit()

        return True

    except sqlite3.Error as error:
        if should_log:
            print("Error while creating the SQLite table:", error)
        return False


def test_llm_sql(llm_response, should_log=True):
    """Runs a suite of SQLite tests"""
    try:
        conn = create_connection()
        cursor = conn.cursor()

        create_response = test_create(conn, cursor, llm_response.create, should_log=should_log)

        select_response = test_select(conn, cursor, llm_response.select, should_log=should_log)

        if conn:
            close_connection(conn)

        if create_response is not True:
            return False

        elif select_response is not True:
            return False

        else:
            return True

    except sqlite3.Error as error:
        if should_log:
            print("Error while creating a sqlite table", error)
        return False
```

```python
# Viewing CREATE and SELECT sqls returned by GPT

test_query = LLMResponse.model_validate_json(content)
print(f"CREATE SQL is: {test_query.create}")
print(f"SELECT SQL is: {test_query.select}")
```

```text
CREATE SQL is: CREATE TABLE DepartmentHeads (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT,
    department VARCHAR(100)
);
SELECT SQL is: SELECT COUNT(*) AS NumberOfHeadsOlderThan56 
FROM DepartmentHeads 
WHERE age > 56;
```

```python
# Testing the CREATE and SELECT sqls are valid (we expect this to be succesful)

test_llm_sql(test_query)
```

```text
Testing create query: CREATE TABLE DepartmentHeads (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT,
    department VARCHAR(100)
);
Testing select query: SELECT COUNT(*) AS NumberOfHeadsOlderThan56 
FROM DepartmentHeads 
WHERE age > 56;
Result of query: [(0,)]
```

```text
True
```

```python
# Again we'll perform a negative test to confirm that a failing SELECT will return an error.

test_failure_query = '{"create": "CREATE TABLE departments (id INT, name VARCHAR(255), head_of_department VARCHAR(255))", "select": "SELECT COUNT(*) FROM departments WHERE age > 56"}'
test_failure_query = LLMResponse.model_validate_json(test_failure_query)
test_llm_sql(test_failure_query)
```

```text
Testing create query: CREATE TABLE departments (id INT, name VARCHAR(255), head_of_department VARCHAR(255))
Testing select query: SELECT COUNT(*) FROM departments WHERE age > 56
Error while executing select query: no such column: age
```

```text
False
```

### Using an LLM to evaluate relevancy

Next, we **evaluate** whether the generated SQL actually answers the user's question. This test will be performed by `gpt-4o-mini`, and will assess how **relevant** the produced SQL query is when compared to the initial user request.

This is a simple example which adapts an approach outlined in the [G-Eval paper](https://arxiv.org/abs/2303.16634), and tested in one of our other [cookbooks](https://github.com/openai/openai-cookbook/blob/main/examples/evaluation/How_to_eval_abstractive_summarization.ipynb).

```python
EVALUATION_MODEL = "gpt-4o-mini"

EVALUATION_PROMPT_TEMPLATE = """
You will be given one summary written for an article. Your task is to rate the summary on one metric.
Please make sure you read and understand these instructions very carefully. 
Please keep this document open while reviewing, and refer to it as needed.

Evaluation Criteria:

{criteria}

Evaluation Steps:

{steps}

Example:

Request:

{request}

Queries:

{queries}

Evaluation Form (scores ONLY):

- {metric_name}
"""

# Relevance

RELEVANCY_SCORE_CRITERIA = """
Relevance(1-5) - review of how relevant the produced SQL queries are to the original question. \
The queries should contain all points highlighted in the user's request. \
Annotators were instructed to penalize queries which contained redundancies and excess information.
"""

RELEVANCY_SCORE_STEPS = """
1. Read the request and the queries carefully.
2. Compare the queries to the request document and identify the main points of the request.
3. Assess how well the queries cover the main points of the request, and how much irrelevant or redundant information it contains.
4. Assign a relevance score from 1 to 5.
"""
```

```python
def get_geval_score(
    criteria: str, steps: str, request: str, queries: str, metric_name: str
):
    """Given evaluation criteria and an observation, this function uses EVALUATION GPT to evaluate the observation against those criteria.
"""
    prompt = EVALUATION_PROMPT_TEMPLATE.format(
        criteria=criteria,
        steps=steps,
        request=request,
        queries=queries,
        metric_name=metric_name,
    )
    response = client.chat.completions.create(
        model=EVALUATION_MODEL,
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
        max_tokens=5,
        top_p=1,
        frequency_penalty=0,
        presence_penalty=0,
    )
    return response.choices[0].message.content
```

```python
# Test out evaluation on a few records

evaluation_results = []

for x,y in sql_df.head(3).iterrows():
    score = get_geval_score(
        RELEVANCY_SCORE_CRITERIA,
        RELEVANCY_SCORE_STEPS,
        y['question'],
        y['context'] + '\n' + y['answer'],'relevancy'
    )
    evaluation_results.append((y['question'],y['context'] + '\n' + y['answer'],score))
```

```python
for result in evaluation_results:
    print(f"User Question \t: {result[0]}")
    print(f"CREATE SQL Returned \t: {result[1].splitlines()[0]}")
    print(f"SELECT SQL Returned \t: {result[1].splitlines()[1]}")
    print(f"{result[2]}")
    print("*" * 20)
```

```text
User Question 	: How many heads of the departments are older than 56 ?
CREATE SQL Returned 	: CREATE TABLE head (age INTEGER)
SELECT SQL Returned 	: SELECT COUNT(*) FROM head WHERE age > 56
5
********************
User Question 	: List the name, born state and age of the heads of departments ordered by age.
CREATE SQL Returned 	: CREATE TABLE head (name VARCHAR, born_state VARCHAR, age VARCHAR)
SELECT SQL Returned 	: SELECT name, born_state, age FROM head ORDER BY age
4
********************
User Question 	: List the creation year, name and budget of each department.
CREATE SQL Returned 	: CREATE TABLE department (creation VARCHAR, name VARCHAR, budget_in_billions VARCHAR)
SELECT SQL Returned 	: SELECT creation, name, budget_in_billions FROM department
4
********************
```

## Evaluation

We will test these functions in combination including our unit test and evaluations to test out two system prompts.

Each iteration of input/output and scores should be stored as a **run**. Optionally you can add GPT-4 annotation within your evaluations or as a separate step to review an entire run and highlight the reasons for errors.

For this example, the second system prompt will include an extra line of clarification, so we can assess the impact of this for both SQL validity and quality of solution.

### Building the test framework

We want to build a function, `test_system_prompt`, which will run our unit tests and evaluation against a given system prompt.

```python
def execute_unit_tests(input_df, output_list, system_prompt):
    """Unit testing function that takes in a dataframe and appends test results to an output_list."""

    for x, y in tqdm(input_df.iterrows(), total=len(input_df)):
        model_response = get_response(system_prompt, y['question'])

        format_valid = test_valid_schema(model_response)

        try:
            test_query = LLMResponse.model_validate_json(model_response)
            # Avoid logging since we're executing many rows at once
            sql_valid = test_llm_sql(test_query, should_log=False)
        except:
            sql_valid = False

        output_list.append((y['question'], model_response, format_valid, sql_valid))
        
def evaluate_row(row):
    """Simple evaluation function to categorize unit testing results.
    
    If the format or SQL are flagged it returns a label, otherwise it is correct"""
    if row['format'] is False:
        return 'Format incorrect'
    elif row['sql'] is False:
        return 'SQL incorrect'
    else:
        return 'SQL correct'

def test_system_prompt(test_df, system_prompt):
    # Execute unit tests and capture results
    results = []
    execute_unit_tests(
        input_df=test_df,
        output_list=results,
        system_prompt=system_prompt
    )
    
    results_df = pd.DataFrame(results)
    results_df.columns = ['question','response','format','sql']
    
    # Use `apply` to calculate the geval score and unit test evaluation
    # for each generated response
    results_df['evaluation_score'] = results_df.apply(
        lambda x: get_geval_score(
            RELEVANCY_SCORE_CRITERIA,
            RELEVANCY_SCORE_STEPS,
            x['question'],
            x['response'],
            'relevancy'
        ),
        axis=1
    )
    results_df['unit_test_evaluation'] = results_df.apply(
        lambda x: evaluate_row(x),
        axis=1
    )
    return results_df
```

### System Prompt 1

The system under test is the first system prompt as shown below.  This `run` will generate responses for this system prompt and evaluate the responses using the functions we've created so far.

```python
system_prompt = """Translate this natural language request into a JSON object
containing two SQL queries.

The first query should be a CREATE statement for a table answering the user's
request, while the second should be a SELECT query answering their question. 
"""

# Select 50 unseen queries to test this one
test_df = sql_df.tail(50)

results_df = test_system_prompt(test_df, system_prompt)
```

```text
  0%|          | 0/50 [00:00<?, ?it/s]
```

We can now group the outcomes of:
* the **unit tests**, which test the structure of response; and
* the **evaluation**, which checks if the SQL is syntactically correct.

```python
results_df['unit_test_evaluation'].value_counts()
```

```text
unit_test_evaluation
SQL correct      46
SQL incorrect     4
Name: count, dtype: int64
```

```python
results_df['evaluation_score'].value_counts()
```

```text
evaluation_score
5    33
4    16
3     1
Name: count, dtype: int64
```

### System Prompt 2

We now use a new system prompt to run same unit test and evaluation.

```python
system_prompt_2 = """Translate this natural language request into a JSON
object containing two SQL queries.

The first query should be a CREATE statement for a table answering the user's
request, while the second should be a SELECT query answering their question.

Ensure the SQL is always generated on one line, never use \\n to separate rows."""


results_2_df = test_system_prompt(test_df, system_prompt)
```

```text
  0%|          | 0/50 [00:00<?, ?it/s]
```

As above, we can group the unit test and evaluation results.

```python
results_2_df['unit_test_evaluation'].value_counts()
```

```text
unit_test_evaluation
SQL correct      44
SQL incorrect     6
Name: count, dtype: int64
```

```python
results_2_df['evaluation_score'].value_counts()
```

```text
evaluation_score
5    34
4    15
3     1
Name: count, dtype: int64
```

## Reporting

We'll make a simple dataframe to store and display the run performance - this is where you can use tools like Weights & Biases Prompts or Gantry to store the results for analytics on your different iterations.

```python
results_df['run'] = 1
results_df['Evaluating Model'] = 'gpt-4'

results_2_df['run'] = 2
results_2_df['Evaluating Model'] = 'gpt-4'

run_df = pd.concat([results_df,results_2_df])
run_df.head()
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>question</th>
      <th>response</th>
      <th>format</th>
      <th>sql</th>
      <th>evaluation_score</th>
      <th>unit_test_evaluation</th>
      <th>run</th>
      <th>Evaluating Model</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>What venue did the parntership of shoaib malik...</td>
      <td>{"create":"CREATE TABLE cricket_partnerships (...</td>
      <td>True</td>
      <td>True</td>
      <td>5</td>
      <td>SQL correct</td>
      <td>1</td>
      <td>gpt-4</td>
    </tr>
    <tr>
      <th>1</th>
      <td>What venue did the partnership of herschelle g...</td>
      <td>{"create":"CREATE TABLE CricketPartnerships (\...</td>
      <td>True</td>
      <td>True</td>
      <td>5</td>
      <td>SQL correct</td>
      <td>1</td>
      <td>gpt-4</td>
    </tr>
    <tr>
      <th>2</th>
      <td>What is the number Played that has 310 Points ...</td>
      <td>{"create":"CREATE TABLE game_stats (\n    numb...</td>
      <td>True</td>
      <td>True</td>
      <td>5</td>
      <td>SQL correct</td>
      <td>1</td>
      <td>gpt-4</td>
    </tr>
    <tr>
      <th>3</th>
      <td>What Losing bonus has a Points against of 588?</td>
      <td>{"create":"CREATE TABLE BonusInfo (\n    id IN...</td>
      <td>True</td>
      <td>True</td>
      <td>5</td>
      <td>SQL correct</td>
      <td>1</td>
      <td>gpt-4</td>
    </tr>
    <tr>
      <th>4</th>
      <td>What Tries against has a Losing bonus of 7?</td>
      <td>{"create":"CREATE TABLE matches (\n    id SERI...</td>
      <td>True</td>
      <td>True</td>
      <td>5</td>
      <td>SQL correct</td>
      <td>1</td>
      <td>gpt-4</td>
    </tr>
  </tbody>
</table>
</div>

#### Plotting unit test results

We can create a simple bar chart to visualise the results of unit tests for both runs.

```python
unittest_df_pivot = pd.pivot_table(
    run_df,
    values='format',
    index=['run','unit_test_evaluation'],
    aggfunc='count'
)
unittest_df_pivot.columns = ['Number of records']
unittest_df_pivot
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>Number of records</th>
    </tr>
    <tr>
      <th>run</th>
      <th>unit_test_evaluation</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">1</th>
      <th>SQL correct</th>
      <td>46</td>
    </tr>
    <tr>
      <th>SQL incorrect</th>
      <td>4</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">2</th>
      <th>SQL correct</th>
      <td>44</td>
    </tr>
    <tr>
      <th>SQL incorrect</th>
      <td>6</td>
    </tr>
  </tbody>
</table>
</div>

```python
unittest_df_pivot.reset_index(inplace=True)

# Plotting
plt.figure(figsize=(10, 6))

# Set the width of each bar
bar_width = 0.35

# OpenAI brand colors
openai_colors = ['#00D1B2', '#000000']  # Green and Black

# Get unique runs and unit test evaluations
unique_runs = unittest_df_pivot['run'].unique()
unique_unit_test_evaluations = unittest_df_pivot['unit_test_evaluation'].unique()

# Ensure we have enough colors (repeating the pattern if necessary)
colors = openai_colors * (len(unique_runs) // len(openai_colors) + 1)

# Iterate over each run to plot
for i, run in enumerate(unique_runs):
    run_data = unittest_df_pivot[unittest_df_pivot['run'] == run]

    # Position of bars for this run
    positions = np.arange(len(unique_unit_test_evaluations)) + i * bar_width

    plt.bar(positions, run_data['Number of records'], width=bar_width, label=f'Run {run}', color=colors[i])

# Setting the x-axis labels to be the unit test evaluations, centered under the groups
plt.xticks(np.arange(len(unique_unit_test_evaluations)) + bar_width / 2, unique_unit_test_evaluations)

plt.xlabel('Unit Test Evaluation')
plt.ylabel('Number of Records')
plt.title('Unit Test Evaluations vs Number of Records for Each Run')
plt.legend()
plt.show()
```

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/evaluation/how_to_evaluate_llms_for_sql_generation/cell-45-output-0.png)

#### Plotting evaluation results

We can similarly plot the results of the evaluation.

```python
evaluation_df_pivot = pd.pivot_table(
    run_df,
    values='format',
    index=['run','evaluation_score'],
    aggfunc='count'
)
evaluation_df_pivot.columns = ['Number of records']
evaluation_df_pivot
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>Number of records</th>
    </tr>
    <tr>
      <th>run</th>
      <th>evaluation_score</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="3" valign="top">1</th>
      <th>3</th>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>16</td>
    </tr>
    <tr>
      <th>5</th>
      <td>33</td>
    </tr>
    <tr>
      <th rowspan="3" valign="top">2</th>
      <th>3</th>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>15</td>
    </tr>
    <tr>
      <th>5</th>
      <td>34</td>
    </tr>
  </tbody>
</table>
</div>

```python
# Reset index without dropping the 'run' and 'evaluation_score' columns
evaluation_df_pivot.reset_index(inplace=True)

# Plotting
plt.figure(figsize=(10, 6))

bar_width = 0.35

# OpenAI brand colors
openai_colors = ['#00D1B2', '#000000']  # Green, Black

# Identify unique runs and evaluation scores
unique_runs = evaluation_df_pivot['run'].unique()
unique_evaluation_scores = evaluation_df_pivot['evaluation_score'].unique()

# Repeat colors if there are more runs than colors
colors = openai_colors * (len(unique_runs) // len(openai_colors) + 1)

for i, run in enumerate(unique_runs):
    # Select rows for this run only
    run_data = evaluation_df_pivot[evaluation_df_pivot['run'] == run].copy()
    
    # Ensure every 'evaluation_score' is present
    run_data.set_index('evaluation_score', inplace=True)
    run_data = run_data.reindex(unique_evaluation_scores, fill_value=0)
    run_data.reset_index(inplace=True)
    
    # Plot each bar
    positions = np.arange(len(unique_evaluation_scores)) + i * bar_width
    plt.bar(
        positions,
        run_data['Number of records'],
        width=bar_width,
        label=f'Run {run}',
        color=colors[i]
    )

# Configure the x-axis to show evaluation scores under the grouped bars
plt.xticks(np.arange(len(unique_evaluation_scores)) + bar_width / 2, unique_evaluation_scores)

plt.xlabel('Evaluation Score')
plt.ylabel('Number of Records')
plt.title('Evaluation Scores vs Number of Records for Each Run')
plt.legend()
plt.show()
```

![](https://developers.openai.com/cookbook/assets/notebook-outputs/examples/evaluation/how_to_evaluate_llms_for_sql_generation/cell-48-output-0.png)

## Conclusion

Now you have a framework to test SQL generation using LLMs, and with some tweaks this approach can be extended to many other code generation use cases. With GPT-4 and engaged human labellers you can aim to automate the evaluation of these test cases, making an iterative loop where new examples are added to the test set and this structure detects any performance regressions. 

We hope you find this useful, and please supply any feedback.