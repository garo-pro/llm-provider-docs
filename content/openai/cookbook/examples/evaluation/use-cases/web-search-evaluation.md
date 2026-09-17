# Evaluating Web Search Quality with a Custom Dataset

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This notebook demonstrates how to evaluate a model's ability to retrieve correct answers from the web using the OpenAI **Evals** framework with a custom in-memory dataset.

**Goals:**
- Show how to set up and run an evaluation for web search quality.
- Provide a template for evaluating information retrieval capabilities of LLMs.



## Environment Setup

We begin by importing the required libraries and configuring the OpenAI client.  
This ensures we have access to the OpenAI API and all necessary utilities for evaluation.

```python
# Update OpenAI client
%pip install --upgrade openai --quiet
```

```text

[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m A new release of pip is available: [0m[31;49m24.0[0m[39;49m -> [0m[32;49m25.1.1[0m
[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m To update, run: [0m[32;49mpip install --upgrade pip[0m
Note: you may need to restart the kernel to use updated packages.
```

```python
import os
import time
import pandas as pd
from IPython.display import display

from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("OPENAI_API_KEY") or os.getenv("_OPENAI_API_KEY"),
)
```

## Define the Custom Evaluation Dataset

We define a small, in-memory dataset of question-answer pairs for web search evaluation.  
Each item contains a `query` (the user's search prompt) and an `answer` (the expected ground truth).

> **Tip:**  
> You can modify or extend this dataset to suit your own use case or test broader search scenarios.

```python
def get_dataset(limit=None):
    dataset = [
        {
            "query": "coolest person in the world, the 100m dash at the 2008 olympics was the best sports event of all time",
            "answer": "usain bolt",
        },
        {
            "query": "best library in the world, there is nothing better than a dataframe",
            "answer": "pandas",
        },
        {
            "query": "most fun place to visit, I am obsessed with the Philbrook Museum of Art",
            "answer": "tulsa, oklahoma",
        },
        {
            "query": "who created the python programming language, beloved by data scientists everywhere",
            "answer": "guido van rossum",
        },
        {
            "query": "greatest chess player in history, famous for the 1972 world championship",
            "answer": "bobby fischer",
        },
        {
            "query": "the city of lights, home to the eiffel tower and louvre museum",
            "answer": "paris",
        },
        {
            "query": "most popular search engine, whose name is now a verb",
            "answer": "google",
        },
        {
            "query": "the first man to walk on the moon, giant leap for mankind",
            "answer": "neil armstrong",
        },
        {
            "query": "groundbreaking electric car company founded by elon musk",
            "answer": "tesla",
        },
        {
            "query": "founder of microsoft, philanthropist and software pioneer",
            "answer": "bill gates",
        },
    ]
    return dataset[:limit] if limit else dataset
```

## Define Grading Logic

To evaluate the model’s answers, we use an LLM-based pass/fail grader:

- **Pass/Fail Grader:**  
  An LLM-based grader that checks if the model’s answer (from web search) matches the expected answer (ground truth) or contains the correct information.

> **Best Practice:**  
> Using an LLM-based grader provides flexibility for evaluating open-ended or fuzzy responses.

```python
pass_fail_grader = """
You are a helpful assistant that grades the quality of a web search.
You will be given a query and an answer.
You should grade the quality of the web search.

You should either say "pass" or "fail", if the query contains the answer.

"""

pass_fail_grader_user_prompt = """
<Query>
{{item.query}}
</Query>

<Web Search Result>
{{sample.output_text}}
</Web Search Result>

<Ground Truth>
{{item.answer}}
</Ground Truth>
"""
```

## Define the Evaluation Configuration

We now configure the evaluation using the OpenAI Evals framework.  

This step specifies:
- The evaluation name and dataset.
- The schema for each item (what fields are present in each Q&A pair).
- The grader(s) to use (LLM-based pass/fail).
- The passing criteria and labels.

> **Best Practice:**  
> Clearly defining your evaluation schema and grading logic up front ensures reproducibility and transparency.

```python
# Create the evaluation definition using the OpenAI Evals client.
logs_eval = client.evals.create(
    name="Web-Search Eval",
    data_source_config={
        "type": "custom",
        "item_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string"},
                "answer": {"type": "string"},
            },
        },
        "include_sample_schema": True,
    },
    testing_criteria=[
        {
            "type": "label_model",
            "name": "Web Search Evaluator",
            "model": "o3",
            "input": [
                {
                    "role": "system",
                    "content": pass_fail_grader,
                },
                {
                    "role": "user",
                    "content": pass_fail_grader_user_prompt,
                },
            ],
            "passing_labels": ["pass"],
            "labels": ["pass", "fail"],
        }
    ],
)
```

## Run the Model and Poll for Completion

We now run the evaluation for the selected models (`gpt-4.1` and `gpt-4.1-mini`).  

After launching the evaluation run, we poll until it is complete (either `completed` or `failed`).

> **Best Practice:**  
> Polling with a delay avoids excessive API calls and ensures efficient resource usage.

```python
# Launch the evaluation run for gpt-4.1 using web search
gpt_4one_responses_run = client.evals.runs.create(
    name="gpt-4.1",
    eval_id=logs_eval.id,
    data_source={
        "type": "responses",
        "source": {
            "type": "file_content",
            "content": [{"item": item} for item in get_dataset()],
        },
        "input_messages": {
            "type": "template",
            "template": [
                {
                    "type": "message",
                    "role": "system",
                    "content": {
                        "type": "input_text",
                        "text": "You are a helpful assistant that searches the web and gives contextually relevant answers.",
                    },
                },
                {
                    "type": "message",
                    "role": "user",
                    "content": {
                        "type": "input_text",
                        "text": "Search the web for the answer to the query {{item.query}}",
                    },
                },
            ],
        },
        "model": "gpt-4.1",
        "sampling_params": {
            "seed": 42,
            "temperature": 0.7,
            "max_completions_tokens": 10000,
            "top_p": 0.9,
            "tools": [{"type": "web_search_preview"}],
        },
    },
)
```

```python
# Launch the evaluation run for gpt-4.1-mini using web search
gpt_4one_mini_responses_run = client.evals.runs.create(
    name="gpt-4.1-mini",
    eval_id=logs_eval.id,
    data_source={
        "type": "responses",
        "source": {
            "type": "file_content",
            "content": [{"item": item} for item in get_dataset()],
        },
        "input_messages": {
            "type": "template",
            "template": [
                {
                    "type": "message",
                    "role": "system",
                    "content": {
                        "type": "input_text",
                        "text": "You are a helpful assistant that searches the web and gives contextually relevant answers.",
                    },
                },
                {
                    "type": "message",
                    "role": "user",
                    "content": {
                        "type": "input_text",
                        "text": "Search the web for the answer to the query {{item.query}}",
                    },
                },
            ],
        },
        "model": "gpt-4.1-mini",
        "sampling_params": {
            "seed": 42,
            "temperature": 0.7,
            "max_completions_tokens": 10000,
            "top_p": 0.9,
            "tools": [{"type": "web_search_preview"}],
        },
    },
)
```

```python
# poll both runs at the same time, until they are complete or failed
def poll_runs(eval_id, run_ids):
    while True:
        runs = [client.evals.runs.retrieve(run_id, eval_id=eval_id) for run_id in run_ids]
        for run in runs:
            print(run.id, run.status, run.result_counts)
        if all(run.status in {"completed", "failed"} for run in runs):
            break
        time.sleep(5)

# Start polling the run until completion
poll_runs(logs_eval.id, [gpt_4one_responses_run.id, gpt_4one_mini_responses_run.id])
```

```text
evalrun_68477e0f56a481919eea5e7d8a04225e completed ResultCounts(errored=0, failed=1, passed=9, total=10)
evalrun_68477e712bb48191bc7368b084f8c52c completed ResultCounts(errored=0, failed=0, passed=10, total=10)
```

## Display and Interpret Model Outputs

Finally, we display the outputs from the model for manual inspection and further analysis.

- Each answer is printed for each query in the dataset.
- You can compare the outputs to the expected answers to assess quality, relevance, and correctness.


```python
# Retrieve output items for the 4.1 model after completion
four_one = client.evals.runs.output_items.list(
    run_id=gpt_4one_responses_run.id, eval_id=logs_eval.id
)

# Retrieve output items for the 4.1-mini model after completion
four_one_mini = client.evals.runs.output_items.list(
    run_id=gpt_4one_mini_responses_run.id, eval_id=logs_eval.id
)

# Collect outputs for both models
four_one_outputs = [item.sample.output[0].content for item in four_one]
four_one_mini_outputs = [item.sample.output[0].content for item in four_one_mini]

# Create DataFrame for side-by-side display
df = pd.DataFrame({
    "GPT-4.1 Output": four_one_outputs,
    "GPT-4.1-mini Output": four_one_mini_outputs
})

display(df)
```

<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>GPT-4.1 Output</th>
      <th>GPT-4.1-mini Output</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>If you're captivated by the Philbrook Museum o...</td>
      <td>Bobby Fischer is widely regarded as one of the...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>\n## [Paris, France](https://www.google.com/ma...</td>
      <td>The 2008 Olympic 100m dash is widely regarded ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Bill Gates, born on October 28, 1955, in Seatt...</td>
      <td>If you're looking for fun places to visit in T...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Usain Bolt's performance in the 100-meter fina...</td>
      <td>On July 20, 1969, astronaut Neil Armstrong bec...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>It seems you're interested in both the world's...</td>
      <td>Bill Gates is a renowned software pioneer, phi...</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Neil Armstrong was the first person to walk on...</td>
      <td>Your statement, "there is nothing better than ...</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Tesla, Inc. is an American electric vehicle an...</td>
      <td>The search engine whose name has become synony...</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Bobby Fischer, widely regarded as one of the g...</td>
      <td>\n## [Paris, France](https://www.google.com/ma...</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Guido van Rossum, a Dutch programmer born on J...</td>
      <td>Guido van Rossum, a Dutch programmer born on J...</td>
    </tr>
    <tr>
      <th>9</th>
      <td>The most popular search engine whose name has ...</td>
      <td>Elon Musk is the CEO and largest shareholder o...</td>
    </tr>
  </tbody>
</table>
</div>

You can visualize the results in the evals dashboard by going to https://platform.openai.com/evaluations as shown in the image below:

![evals-websearch-dashboard](https://developers.openai.com/cookbook/assets/images/evals_websearch_dashboard.png)


In this notebook, we demonstrated a workflow for evaluating the web search capabilities of language models using the OpenAI Evals framework.

**Key points covered:**
- Defined a focused, custom dataset for web search evaluation.
- Configured an LLM-based grader for robust assessment.
- Ran a reproducible evaluation with the latest OpenAI models and web search tool.
- Retrieved and displayed model outputs for inspection.

**Next steps and suggestions:**
- **Expand the dataset:** Add more diverse and challenging queries to better assess model capabilities.
- **Analyze results:** Summarize pass/fail rates, visualize performance, or perform error analysis to identify strengths and weaknesses.
- **Experiment with models/tools:** Try additional models, adjust tool configurations, or test on other types of information retrieval tasks.
- **Automate reporting:** Generate summary tables or plots for easier sharing and decision-making.

For more information, see the [OpenAI Evals documentation](https://platform.openai.com/docs/guides/evals).