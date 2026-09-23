# Tool Evaluation with OpenAI Evals

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

This cookbook shows how to **measure and improve a model’s ability to extract structured information from source code** with tool evaluation. In this case, the set of *symbols* (functions, classes, methods, and variables) defined in Python files.  

## Setup<a name="Setup"></a>

Install the latest **openai** Python package ≥ 1.14.0 and set your `OPENAI_API_KEY` environment variable.  If you also want to evaluate an *assistant with tools*, enable the *Assistants v2 beta* in your account.

```bash
pip install --upgrade openai
export OPENAI_API_KEY=sk‑...
```
Below we import the SDK, create a client, and define a helper that builds a small dataset from files inside the **openai** package itself.

```python
%pip install --upgrade openai pandas jinja2 rich --quiet

import os
import time
import openai
from rich import print

client = openai.OpenAI(
    api_key=os.getenv("OPENAI_API_KEY") or os.getenv("_OPENAI_API_KEY"),
)
```

```text

[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m A new release of pip is available: [0m[31;49m24.0[0m[39;49m -> [0m[32;49m25.1.1[0m
[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m To update, run: [0m[32;49mpip install --upgrade pip[0m
Note: you may need to restart the kernel to use updated packages.
```

### Dataset factory & grading rubric
* `get_dataset` builds a small in-memory dataset by reading several SDK files.
* `structured_output_grader` defines a detailed evaluation rubric. 
* `sampled.output_tools[0].function.arguments.symbols` specifies the extracted symbols from the code file based on the tool invocation.
* `client.evals.create(...)` registers the eval with the platform.

```python
def get_dataset(limit=None):
    openai_sdk_file_path = os.path.dirname(openai.__file__)

    file_paths = [
        os.path.join(openai_sdk_file_path, "resources", "evals", "evals.py"),
        os.path.join(openai_sdk_file_path, "resources", "responses", "responses.py"),
        os.path.join(openai_sdk_file_path, "resources", "images.py"),
        os.path.join(openai_sdk_file_path, "resources", "embeddings.py"),
        os.path.join(openai_sdk_file_path, "resources", "files.py"),
    ]

    items = []
    for file_path in file_paths:
        items.append({"input": open(file_path, "r").read()})
    if limit:
        return items[:limit]
    return items


structured_output_grader = """
You are a helpful assistant that grades the quality of extracted information from a code file.
You will be given a code file and a list of extracted information.
You should grade the quality of the extracted information.

You should grade the quality on a scale of 1 to 7.
You should apply the following criteria, and calculate your score as follows:
You should first check for completeness on a scale of 1 to 7.
Then you should apply a quality modifier.

The quality modifier is a multiplier from 0 to 1 that you multiply by the completeness score.
If there is 100% coverage for completion and it is all high quality, then you would return 7*1.
If there is 100% coverage for completion but it is all low quality, then you would return 7*0.5.
etc.
"""

structured_output_grader_user_prompt = """
<Code File>
{{item.input}}
</Code File>

<Extracted Information>
{{sample.output_tools[0].function.arguments.symbols}}
</Extracted Information>
"""
```

### Evals Creation

Here we create an eval that will be used to evaluate the quality of extracted information from code files.


```python
logs_eval = client.evals.create(
    name="Code QA Eval",
    data_source_config={
        "type": "custom",
        "item_schema": {"type": "object", "properties": {"input": {"type": "string"}}},
        "include_sample_schema": True,
    },
    testing_criteria=[
        {
            "type": "score_model",
            "name": "General Evaluator",
            "model": "o3",
            "input": [
                {"role": "system", "content": structured_output_grader},
                {"role": "user", "content": structured_output_grader_user_prompt},
            ],
            "range": [1, 7],
            "pass_threshold": 5.0,
        }
    ],
)

symbol_tool = {
    "name": "extract_symbols",
    "description": "Extract the symbols from the code file",
    "parameters": {
        "type": "object",
        "properties": {
            "symbols": {
                "type": "array",
                "description": "A list of symbols extracted from Python code.",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string", "description": "The name of the symbol."},
                        "symbol_type": {"type": "string", "description": "The type of the symbol, e.g., variable, function, class."},
                    },
                    "required": ["name", "symbol_type"],
                    "additionalProperties": False,
                },
            }
        },
        "required": ["symbols"],
        "additionalProperties": False,
    },
}
```

### Kick off model runs
Here we launch two runs against the same eval: one that calls the **Completions** endpoint, and one that calls the **Responses** endpoint.

```python
gpt_4one_completions_run = client.evals.runs.create(
    name="gpt-4.1",
    eval_id=logs_eval.id,
    data_source={
        "type": "completions",
        "source": {"type": "file_content", "content": [{"item": item} for item in get_dataset(limit=1)]},
        "input_messages": {
            "type": "template",
            "template": [
                {"type": "message", "role": "system", "content": {"type": "input_text", "text": "You are a helpful assistant."}},
                {"type": "message", "role": "user", "content": {"type": "input_text", "text": "Extract the symbols from the code file {{item.input}}"}},
            ],
        },
        "model": "gpt-4.1",
        "sampling_params": {
            "seed": 42,
            "temperature": 0.7,
            "max_completions_tokens": 10000,
            "top_p": 0.9,
            "tools": [{"type": "function", "function": symbol_tool}],
        },
    },
)

gpt_4one_responses_run = client.evals.runs.create(
    name="gpt-4.1-mini",
    eval_id=logs_eval.id,
    data_source={
        "type": "responses",
        "source": {"type": "file_content", "content": [{"item": item} for item in get_dataset(limit=1)]},
        "input_messages": {
            "type": "template",
            "template": [
                {"type": "message", "role": "system", "content": {"type": "input_text", "text": "You are a helpful assistant."}},
                {"type": "message", "role": "user", "content": {"type": "input_text", "text": "Extract the symbols from the code file {{item.input}}"}},
            ],
        },
        "model": "gpt-4.1-mini",
        "sampling_params": {
            "seed": 42,
            "temperature": 0.7,
            "max_completions_tokens": 10000,
            "top_p": 0.9,
            "tools": [{"type": "function", **symbol_tool}],
        },
    },
)
```

### Utility Poller

We create a utility poller that will be used to poll for the results of the eval runs.

```python
def poll_runs(eval_id, run_ids):
    # poll both runs at the same time, until they are complete or failed
    while True:
        runs = [client.evals.runs.retrieve(run_id, eval_id=eval_id) for run_id in run_ids]
        for run in runs:
            print(run.id, run.status, run.result_counts)
        if all(run.status in ("completed", "failed") for run in runs):
            break
        time.sleep(5)


poll_runs(logs_eval.id, [gpt_4one_completions_run.id, gpt_4one_responses_run.id])
```

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">evalrun_6848e2269570819198b757fe12b979da completed
<span style="color: #800080; text-decoration-color: #800080; font-weight: bold">ResultCounts</span><span style="font-weight: bold">(</span><span style="color: #808000; text-decoration-color: #808000">errored</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>, <span style="color: #808000; text-decoration-color: #808000">failed</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #808000; text-decoration-color: #808000">passed</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>, <span style="color: #808000; text-decoration-color: #808000">total</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span><span style="font-weight: bold">)</span>
</pre>

<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,&#x27;DejaVu Sans Mono&#x27;,consolas,&#x27;Courier New&#x27;,monospace">evalrun_6848e227d3a481918a9b970c897b5998 completed
<span style="color: #800080; text-decoration-color: #800080; font-weight: bold">ResultCounts</span><span style="font-weight: bold">(</span><span style="color: #808000; text-decoration-color: #808000">errored</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>, <span style="color: #808000; text-decoration-color: #808000">failed</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span>, <span style="color: #808000; text-decoration-color: #808000">passed</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">0</span>, <span style="color: #808000; text-decoration-color: #808000">total</span>=<span style="color: #008080; text-decoration-color: #008080; font-weight: bold">1</span><span style="font-weight: bold">)</span>
</pre>

```python

### Get Output
completions_output = client.evals.runs.output_items.list(
    run_id=gpt_4one_completions_run.id, eval_id=logs_eval.id
)

responses_output = client.evals.runs.output_items.list(
    run_id=gpt_4one_responses_run.id, eval_id=logs_eval.id
)
```

### Inspecting results<a name="Inspecting-results"></a>

For both completions and responses, we print the *symbols* dictionary that the model returned. You can diff this against the reference answer or compute precision / recall.

```python
import json
import pandas as pd
from IPython.display import display, HTML

def extract_symbols(output_list):
    symbols_list = []
    for item in output_list:
        try:
            args = item.sample.output[0].tool_calls[0]["function"]["arguments"]
            symbols = json.loads(args)["symbols"]
            symbols_list.append(symbols)
        except Exception as e:
            symbols_list.append([{"error": str(e)}])
    return symbols_list

completions_symbols = extract_symbols(completions_output)
responses_symbols = extract_symbols(responses_output)

def symbols_to_html_table(symbols):
    if symbols and isinstance(symbols, list):
        df = pd.DataFrame(symbols)
        return (
            df.style
            .set_properties(**{
                'white-space': 'pre-wrap',
                'word-break': 'break-word',
                'padding': '2px 6px',
                'border': '1px solid #C3E7FA',
                'font-size': '0.92em',
                'background-color': '#FDFEFF'
            })
            .set_table_styles([{
                'selector': 'th',
                'props': [
                    ('font-size', '0.95em'),
                    ('background-color', '#1CA7EC'),
                    ('color', '#fff'),
                    ('border-bottom', '1px solid #18647E'),
                    ('padding', '2px 6px')
                ]
            }])
            .hide(axis='index')
            .to_html()
        )
    return f"<div style='padding:4px 0;color:#D9534F;font-style:italic;font-size:0.9em'>{str(symbols)}</div>"

table_rows = []
max_len = max(len(completions_symbols), len(responses_symbols))
for i in range(max_len):
    c_html = symbols_to_html_table(completions_symbols[i]) if i < len(completions_symbols) else ""
    r_html = symbols_to_html_table(responses_symbols[i]) if i < len(responses_symbols) else ""
    table_rows.append(f"""
      <tr style="height:1.2em;">
          <td style="vertical-align:top; background:#F6F8FA; border-right:1px solid #E3E3E3; padding:2px 4px;">{c_html}</td>
          <td style="vertical-align:top; background:#F6F8FA; padding:2px 4px;">{r_html}</td>
      </tr>
    """)

table_html = f"""
<div style="margin-bottom:0.5em;margin-top:0.2em;">
  <h4 style="color:#1CA7EC;font-weight:600;letter-spacing:0.5px;
     text-shadow:0 1px 2px rgba(0,0,0,0.06), 0 0px 0px #fff;font-size:1.05em;margin:0 0 0.35em 0;">
    Completions vs Responses Output Symbols
  </h4>
  <table style="border-collapse:separate;border-spacing:0 0.2em;width:100%;border-radius:5px;overflow:hidden;box-shadow:0 1px 7px #BEE7FA22;">
    <thead>
      <tr style="height:1.4em;">
              <th style="width:50%;background:#323C50;color:#fff;font-size:1em;padding:6px 10px;border-bottom:2px solid #1CA7EC;text-align:center;">Completions Output</th>
      <th style="width:50%;background:#323C50;color:#fff;font-size:1em;padding:6px 10px;border-bottom:2px solid #1CA7EC;text-align:center;">Responses Output</th>
      </tr>
    </thead>
    <tbody>
      {''.join(table_rows)}
    </tbody>
  </table>
</div>
"""

display(HTML(table_html))
```

<div style="margin-bottom:0.5em;margin-top:0.2em;">
  <h4 style="color:#1CA7EC;font-weight:600;letter-spacing:0.5px;
     text-shadow:0 1px 2px rgba(0,0,0,0.06), 0 0px 0px #fff;font-size:1.05em;margin:0 0 0.35em 0;">
    Completions vs Responses Output Symbols
  </h4>
  <table style="border-collapse:separate;border-spacing:0 0.2em;width:100%;border-radius:5px;overflow:hidden;box-shadow:0 1px 7px #BEE7FA22;">
    <thead>
      <tr style="height:1.4em;">
              <th style="width:50%;background:#323C50;color:#fff;font-size:1em;padding:6px 10px;border-bottom:2px solid #1CA7EC;text-align:center;">Completions Output</th>
      <th style="width:50%;background:#323C50;color:#fff;font-size:1em;padding:6px 10px;border-bottom:2px solid #1CA7EC;text-align:center;">Responses Output</th>
      </tr>
    </thead>
    <tbody>
      
      <tr style="height:1.2em;">
          <td style="vertical-align:top; background:#F6F8FA; border-right:1px solid #E3E3E3; padding:2px 4px;">
<table id="T_f295b">
  <thead>
    <tr>
      <th id="T_f295b_level0_col0" class="col_heading level0 col0">name</th>
      <th id="T_f295b_level0_col1" class="col_heading level0 col1">symbol_type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td id="T_f295b_row0_col0" class="data row0 col0">Evals</td>
      <td id="T_f295b_row0_col1" class="data row0 col1">class</td>
    </tr>
    <tr>
      <td id="T_f295b_row1_col0" class="data row1 col0">AsyncEvals</td>
      <td id="T_f295b_row1_col1" class="data row1 col1">class</td>
    </tr>
    <tr>
      <td id="T_f295b_row2_col0" class="data row2 col0">EvalsWithRawResponse</td>
      <td id="T_f295b_row2_col1" class="data row2 col1">class</td>
    </tr>
    <tr>
      <td id="T_f295b_row3_col0" class="data row3 col0">AsyncEvalsWithRawResponse</td>
      <td id="T_f295b_row3_col1" class="data row3 col1">class</td>
    </tr>
    <tr>
      <td id="T_f295b_row4_col0" class="data row4 col0">EvalsWithStreamingResponse</td>
      <td id="T_f295b_row4_col1" class="data row4 col1">class</td>
    </tr>
    <tr>
      <td id="T_f295b_row5_col0" class="data row5 col0">AsyncEvalsWithStreamingResponse</td>
      <td id="T_f295b_row5_col1" class="data row5 col1">class</td>
    </tr>
    <tr>
      <td id="T_f295b_row6_col0" class="data row6 col0">__all__</td>
      <td id="T_f295b_row6_col1" class="data row6 col1">variable</td>
    </tr>
  </tbody>
</table>
</td>
          <td style="vertical-align:top; background:#F6F8FA; padding:2px 4px;">
<table id="T_c1589">
  <thead>
    <tr>
      <th id="T_c1589_level0_col0" class="col_heading level0 col0">name</th>
      <th id="T_c1589_level0_col1" class="col_heading level0 col1">symbol_type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td id="T_c1589_row0_col0" class="data row0 col0">Evals</td>
      <td id="T_c1589_row0_col1" class="data row0 col1">class</td>
    </tr>
    <tr>
      <td id="T_c1589_row1_col0" class="data row1 col0">runs</td>
      <td id="T_c1589_row1_col1" class="data row1 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row2_col0" class="data row2 col0">with_raw_response</td>
      <td id="T_c1589_row2_col1" class="data row2 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row3_col0" class="data row3 col0">with_streaming_response</td>
      <td id="T_c1589_row3_col1" class="data row3 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row4_col0" class="data row4 col0">create</td>
      <td id="T_c1589_row4_col1" class="data row4 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row5_col0" class="data row5 col0">retrieve</td>
      <td id="T_c1589_row5_col1" class="data row5 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row6_col0" class="data row6 col0">update</td>
      <td id="T_c1589_row6_col1" class="data row6 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row7_col0" class="data row7 col0">list</td>
      <td id="T_c1589_row7_col1" class="data row7 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row8_col0" class="data row8 col0">delete</td>
      <td id="T_c1589_row8_col1" class="data row8 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row9_col0" class="data row9 col0">AsyncEvals</td>
      <td id="T_c1589_row9_col1" class="data row9 col1">class</td>
    </tr>
    <tr>
      <td id="T_c1589_row10_col0" class="data row10 col0">runs</td>
      <td id="T_c1589_row10_col1" class="data row10 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row11_col0" class="data row11 col0">with_raw_response</td>
      <td id="T_c1589_row11_col1" class="data row11 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row12_col0" class="data row12 col0">with_streaming_response</td>
      <td id="T_c1589_row12_col1" class="data row12 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row13_col0" class="data row13 col0">create</td>
      <td id="T_c1589_row13_col1" class="data row13 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row14_col0" class="data row14 col0">retrieve</td>
      <td id="T_c1589_row14_col1" class="data row14 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row15_col0" class="data row15 col0">update</td>
      <td id="T_c1589_row15_col1" class="data row15 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row16_col0" class="data row16 col0">list</td>
      <td id="T_c1589_row16_col1" class="data row16 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row17_col0" class="data row17 col0">delete</td>
      <td id="T_c1589_row17_col1" class="data row17 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row18_col0" class="data row18 col0">EvalsWithRawResponse</td>
      <td id="T_c1589_row18_col1" class="data row18 col1">class</td>
    </tr>
    <tr>
      <td id="T_c1589_row19_col0" class="data row19 col0">__init__</td>
      <td id="T_c1589_row19_col1" class="data row19 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row20_col0" class="data row20 col0">runs</td>
      <td id="T_c1589_row20_col1" class="data row20 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row21_col0" class="data row21 col0">AsyncEvalsWithRawResponse</td>
      <td id="T_c1589_row21_col1" class="data row21 col1">class</td>
    </tr>
    <tr>
      <td id="T_c1589_row22_col0" class="data row22 col0">__init__</td>
      <td id="T_c1589_row22_col1" class="data row22 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row23_col0" class="data row23 col0">runs</td>
      <td id="T_c1589_row23_col1" class="data row23 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row24_col0" class="data row24 col0">EvalsWithStreamingResponse</td>
      <td id="T_c1589_row24_col1" class="data row24 col1">class</td>
    </tr>
    <tr>
      <td id="T_c1589_row25_col0" class="data row25 col0">__init__</td>
      <td id="T_c1589_row25_col1" class="data row25 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row26_col0" class="data row26 col0">runs</td>
      <td id="T_c1589_row26_col1" class="data row26 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row27_col0" class="data row27 col0">AsyncEvalsWithStreamingResponse</td>
      <td id="T_c1589_row27_col1" class="data row27 col1">class</td>
    </tr>
    <tr>
      <td id="T_c1589_row28_col0" class="data row28 col0">__init__</td>
      <td id="T_c1589_row28_col1" class="data row28 col1">function</td>
    </tr>
    <tr>
      <td id="T_c1589_row29_col0" class="data row29 col0">runs</td>
      <td id="T_c1589_row29_col1" class="data row29 col1">function</td>
    </tr>
  </tbody>
</table>
</td>
      </tr>
    
    </tbody>
  </table>
</div>

### Visualize Evals Dashboard

You can navigate to the Evals Dashboard in order to visualize the data.


![evals_tool_dashboard](https://developers.openai.com/cookbook/assets/images/evals_tool_dashboard.png)


You can also take a look at the explanation of the failed results in the Evals Dashboard after the run is complete as shown in the image below.

![evals_tool_failed](https://developers.openai.com/cookbook/assets/images/eval_tools_fail.png)




This notebook demonstrated how to use OpenAI Evals to assess and improve a model’s ability to extract structured information from Python code using tool calls. 


OpenAI Evals provides a robust, reproducible framework for evaluating LLMs on structured extraction tasks. By combining clear tool schemas, rigorous grading rubrics, and well-structured datasets, you can measure and improve overall performance.

*For more details, see the [OpenAI Evals documentation](https://platform.openai.com/docs/guides/evals).*