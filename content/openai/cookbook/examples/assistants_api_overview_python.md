# Assistants API Overview (Python SDK)

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

The new [Assistants API](https://platform.openai.com/docs/assistants/overview) is a stateful evolution of our [Chat Completions API](https://platform.openai.com/docs/guides/text-generation/chat-completions-api) meant to simplify the creation of assistant-like experiences, and enable developer access to powerful tools like Code Interpreter and File Search.

![Assistants API Diagram](https://developers.openai.com/cookbook/assets/images/assistants_overview_diagram.png)

## Chat Completions API vs Assistants API

The primitives of the **Chat Completions API** are `Messages`, on which you perform a `Completion` with a `Model` (`gpt-4o`, `gpt-4o-mini`, etc). It is lightweight and powerful, but inherently stateless, which means you have to manage conversation state, tool definitions, retrieval documents, and code execution manually.

The primitives of the **Assistants API** are

- `Assistants`, which encapsulate a base model, instructions, tools, and (context) documents,
- `Threads`, which represent the state of a conversation, and
- `Runs`, which power the execution of an `Assistant` on a `Thread`, including textual responses and multi-step tool use.

We'll take a look at how these can be used to create powerful, stateful experiences.


## Setup

### Python SDK

> **Note**
> We've updated our [Python SDK](https://github.com/openai/openai-python) to add support for the Assistants API, so you'll need to update it to the latest version (`1.59.4` at time of writing).


```python
!pip install --upgrade openai
```

```text
Requirement already satisfied: openai in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (1.59.4)
Requirement already satisfied: anyio<5,>=3.5.0 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (3.7.1)
Requirement already satisfied: distro<2,>=1.7.0 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (1.9.0)
Requirement already satisfied: httpx<1,>=0.23.0 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (0.27.0)
Requirement already satisfied: jiter<1,>=0.4.0 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (0.7.0)
Requirement already satisfied: pydantic<3,>=1.9.0 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (2.8.2)
Requirement already satisfied: sniffio in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (1.3.1)
Requirement already satisfied: tqdm>4 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (4.66.4)
Requirement already satisfied: typing-extensions<5,>=4.11 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from openai) (4.12.2)
Requirement already satisfied: idna>=2.8 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from anyio<5,>=3.5.0->openai) (3.7)
Requirement already satisfied: certifi in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from httpx<1,>=0.23.0->openai) (2024.7.4)
Requirement already satisfied: httpcore==1.* in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from httpx<1,>=0.23.0->openai) (1.0.5)
Requirement already satisfied: h11<0.15,>=0.13 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from httpcore==1.*->httpx<1,>=0.23.0->openai) (0.14.0)
Requirement already satisfied: annotated-types>=0.4.0 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from pydantic<3,>=1.9.0->openai) (0.7.0)
Requirement already satisfied: pydantic-core==2.20.1 in /Users/lee.spacagna/myenv/lib/python3.12/site-packages (from pydantic<3,>=1.9.0->openai) (2.20.1)
```

And make sure it's up to date by running:


```python
!pip show openai | grep Version
```

```text
Version: 1.59.4
```

### Pretty Printing Helper


```python
import json

def show_json(obj):
    display(json.loads(obj.model_dump_json()))
```

## Complete Example with Assistants API


### Assistants


The easiest way to get started with the Assistants API is through the [Assistants Playground](https://platform.openai.com/playground).


![Assistants Playground](https://developers.openai.com/cookbook/assets/images/assistants_overview_assistants_playground.png)


Let's begin by creating an assistant! We'll create a Math Tutor just like in our [docs](https://platform.openai.com/docs/assistants/overview).


![Creating New Assistant](https://developers.openai.com/cookbook/assets/images/assistants_overview_new_assistant.png)


You can also create Assistants directly through the Assistants API, like so:


```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY", "<your OpenAI API key if not set as env var>"))


assistant = client.beta.assistants.create(
    name="Math Tutor",
    instructions="You are a personal math tutor. Answer questions briefly, in a sentence or less.",
    model="gpt-4o",
)
show_json(assistant)
```

```text
{'id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'created_at': 1736340398,
 'description': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'metadata': {},
 'model': 'gpt-4o',
 'name': 'Math Tutor',
 'object': 'assistant',
 'tools': [],
 'response_format': 'auto',
 'temperature': 1.0,
 'tool_resources': {'code_interpreter': None, 'file_search': None},
 'top_p': 1.0} 'tools': [],
 'response_format': 'auto',
 'temperature': 1.0,
 'tool_resources': {'code_interpreter': None, 'file_search': None},
 'top_p': 1.0}
```

Regardless of whether you create your Assistant through the Dashboard or with the API, you'll want to keep track of the Assistant ID. This is how you'll refer to your Assistant throughout Threads and Runs.


Next, we'll create a new Thread and add a Message to it. This will hold the state of our conversation, so we don't have re-send the entire message history each time.


### Threads


Create a new thread:


```python
thread = client.beta.threads.create()
show_json(thread)
```

```text
{'id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6',
 'created_at': 1736340398,
 'metadata': {},
 'object': 'thread',
 'tool_resources': {'code_interpreter': None, 'file_search': None}} 'object': 'thread',
 'tool_resources': {'code_interpreter': None, 'file_search': None}}
```

Then add the Message to the thread:


```python
message = client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="I need to solve the equation `3x + 11 = 14`. Can you help me?",
)
show_json(message)
```

```text
{'id': 'msg_1q4Y7ZZ9gIcPoAKSx9UtrrKJ',
 'assistant_id': None,
 'attachments': [],
 'completed_at': None,
 'attachments': [],
 'completed_at': None,
 'content': [{'text': {'annotations': [],
    'value': 'I need to solve the equation `3x + 11 = 14`. Can you help me?'},
   'type': 'text'}],
 'created_at': 1736340400,
 'incomplete_at': None,
 'incomplete_details': None,
 'metadata': {},
 'object': 'thread.message',
 'role': 'user',
 'run_id': None,
 'status': None,
 'thread_id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6'}
```

> **Note**
> Even though you're no longer sending the entire history each time, you will still be charged for the tokens of the entire conversation history with each Run.


### Runs

Notice how the Thread we created is **not** associated with the Assistant we created earlier! Threads exist independently from Assistants, which may be different from what you'd expect if you've used ChatGPT (where a thread is tied to a model/GPT).

To get a completion from an Assistant for a given Thread, we must create a Run. Creating a Run will indicate to an Assistant it should look at the messages in the Thread and take action: either by adding a single response, or using tools.

> **Note**
> Runs are a key difference between the Assistants API and Chat Completions API. While in Chat Completions the model will only ever respond with a single message, in the Assistants API a Run may result in an Assistant using one or multiple tools, and potentially adding multiple messages to the Thread.

To get our Assistant to respond to the user, let's create the Run. As mentioned earlier, you must specify _both_ the Assistant and the Thread.


```python
run = client.beta.threads.runs.create(
    thread_id=thread.id,
    assistant_id=assistant.id,
)
show_json(run)
```

```text
{'id': 'run_qVYsWok6OCjHxkajpIrdHuVP',
 'assistant_id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'cancelled_at': None,
 'completed_at': None,
 'created_at': 1736340403,
 'expires_at': 1736341003,
 'failed_at': None,
 'incomplete_details': None,
 'incomplete_details': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'last_error': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'metadata': {},
 'model': 'gpt-4o',
 'object': 'thread.run',
 'parallel_tool_calls': True,
 'parallel_tool_calls': True,
 'required_action': None,
 'response_format': 'auto',
 'response_format': 'auto',
 'started_at': None,
 'status': 'queued',
 'thread_id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6',
 'tool_choice': 'auto',
 'tools': [],
 'truncation_strategy': {'type': 'auto', 'last_messages': None},
 'usage': None,
 'temperature': 1.0,
 'top_p': 1.0,
 'tool_resources': {}}
```

Unlike creating a completion in the Chat Completions API, **creating a Run is an asynchronous operation**. It will return immediately with the Run's metadata, which includes a `status` that will initially be set to `queued`. The `status` will be updated as the Assistant performs operations (like using tools and adding messages).

To know when the Assistant has completed processing, we can poll the Run in a loop. (Support for streaming is coming soon!) While here we are only checking for a `queued` or `in_progress` status, in practice a Run may undergo a [variety of status changes](https://platform.openai.com/docs/api-reference/runs/object#runs/object-status) which you can choose to surface to the user. (These are called Steps, and will be covered later.)


```python
import time

def wait_on_run(run, thread):
    while run.status == "queued" or run.status == "in_progress":
        run = client.beta.threads.runs.retrieve(
            thread_id=thread.id,
            run_id=run.id,
        )
        time.sleep(0.5)
    return run
```

```python
run = wait_on_run(run, thread)
show_json(run)
```

```text
{'id': 'run_qVYsWok6OCjHxkajpIrdHuVP',
 'assistant_id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'cancelled_at': None,
 'completed_at': 1736340406,
 'created_at': 1736340403,
 'expires_at': None,
 'failed_at': None,
 'incomplete_details': None,
 'incomplete_details': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'last_error': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'metadata': {},
 'model': 'gpt-4o',
 'object': 'thread.run',
 'parallel_tool_calls': True,
 'parallel_tool_calls': True,
 'required_action': None,
 'response_format': 'auto',
 'started_at': 1736340405,
 'status': 'completed',
 'thread_id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6',
 'tool_choice': 'auto',
 'tools': [],
 'truncation_strategy': {'type': 'auto', 'last_messages': None},
 'usage': {'completion_tokens': 35,
  'prompt_tokens': 66,
  'total_tokens': 101,
  'prompt_token_details': {'cached_tokens': 0},
  'completion_tokens_details': {'reasoning_tokens': 0}},
 'temperature': 1.0,
 'top_p': 1.0,
 'tool_resources': {}}
```

### Messages


Now that the Run has completed, we can list the Messages in the Thread to see what got added by the Assistant.


```python
messages = client.beta.threads.messages.list(thread_id=thread.id)
show_json(messages)
```

```text
{'data': [{'id': 'msg_A5eAN6ZAJDmFBOYutEm5DFCy',
   'assistant_id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
   'attachments': [],
   'completed_at': None,
   'content': [{'text': {'annotations': [],
      'value': 'Sure! Subtract 11 from both sides to get \\(3x = 3\\), then divide by 3 to find \\(x = 1\\).'},
     'type': 'text'}],
   'created_at': 1736340405,
   'incomplete_at': None,
   'incomplete_details': None,
   'metadata': {},
   'object': 'thread.message',
   'role': 'assistant',
   'run_id': 'run_qVYsWok6OCjHxkajpIrdHuVP',
   'status': None,
   'thread_id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6'},
  {'id': 'msg_1q4Y7ZZ9gIcPoAKSx9UtrrKJ',
   'assistant_id': None,
   'attachments': [],
   'completed_at': None,
   'attachments': [],
   'completed_at': None,
   'content': [{'text': {'annotations': [],
      'value': 'I need to solve the equation `3x + 11 = 14`. Can you help me?'},
     'type': 'text'}],
   'created_at': 1736340400,
   'incomplete_at': None,
   'incomplete_details': None,
   'metadata': {},
   'object': 'thread.message',
   'role': 'user',
   'run_id': None,
   'status': None,
   'thread_id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6'}],
 'object': 'list',
 'first_id': 'msg_A5eAN6ZAJDmFBOYutEm5DFCy',
 'last_id': 'msg_1q4Y7ZZ9gIcPoAKSx9UtrrKJ',
 'has_more': False}
```

As you can see, Messages are ordered in reverse-chronological order – this was done so the most recent results are always on the first `page` (since results can be paginated). Do keep a look out for this, since this is the opposite order to messages in the Chat Completions API.


Let's ask our Assistant to explain the result a bit further!


```python
# Create a message to append to our thread
message = client.beta.threads.messages.create(
    thread_id=thread.id, role="user", content="Could you explain this to me?"
)

# Execute our run
run = client.beta.threads.runs.create(
    thread_id=thread.id,
    assistant_id=assistant.id,
)

# Wait for completion
wait_on_run(run, thread)

# Retrieve all the messages added after our last user message
messages = client.beta.threads.messages.list(
    thread_id=thread.id, order="asc", after=message.id
)
show_json(messages)
```

```text
{'data': [{'id': 'msg_wSHHvaMnaWktZWsKs6gyoPUB',
   'assistant_id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
   'attachments': [],
   'completed_at': None,
   'content': [{'text': {'annotations': [],
      'value': 'Certainly! To isolate \\(x\\), first subtract 11 from both sides of the equation \\(3x + 11 = 14\\), resulting in \\(3x = 3\\). Then, divide both sides by 3 to solve for \\(x\\), giving you \\(x = 1\\).'},
     'type': 'text'}],
   'created_at': 1736340414,
   'incomplete_at': None,
   'incomplete_details': None,
   'metadata': {},
   'object': 'thread.message',
   'role': 'assistant',
   'run_id': 'run_lJsumsDtPTmdG3Enx2CfYrrq',
   'status': None,
   'thread_id': 'thread_j4dc1TiHPfkviKUHNi4aAsA6'}],
 'object': 'list',
 'first_id': 'msg_wSHHvaMnaWktZWsKs6gyoPUB',
 'last_id': 'msg_wSHHvaMnaWktZWsKs6gyoPUB',
 'has_more': False}
```

This may feel like a lot of steps to get a response back, especially for this simple example. However, you'll soon see how we can add very powerful functionality to our Assistant without changing much code at all!


### Example


Let's take a look at how we could potentially put all of this together. Below is all the code you need to use an Assistant you've created.

Since we've already created our Math Assistant, I've saved its ID in `MATH_ASSISTANT_ID`. I then defined two functions:

- `submit_message`: create a Message on a Thread, then start (and return) a new Run
- `get_response`: returns the list of Messages in a Thread


```python
from openai import OpenAI

MATH_ASSISTANT_ID = assistant.id  # or a hard-coded ID like "asst-..."

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY", "<your OpenAI API key if not set as env var>"))

def submit_message(assistant_id, thread, user_message):
    client.beta.threads.messages.create(
        thread_id=thread.id, role="user", content=user_message
    )
    return client.beta.threads.runs.create(
        thread_id=thread.id,
        assistant_id=assistant_id,
    )


def get_response(thread):
    return client.beta.threads.messages.list(thread_id=thread.id, order="asc")
```

I've also defined a `create_thread_and_run` function that I can reuse (which is actually almost identical to the [`client.beta.threads.create_and_run`](https://platform.openai.com/docs/api-reference/runs/createThreadAndRun) compound function in our API ;) ). Finally, we can submit our mock user requests each to a new Thread.

Notice how all of these API calls are asynchronous operations; this means we actually get async behavior in our code without the use of async libraries! (e.g. `asyncio`)


```python
def create_thread_and_run(user_input):
    thread = client.beta.threads.create()
    run = submit_message(MATH_ASSISTANT_ID, thread, user_input)
    return thread, run


# Emulating concurrent user requests
thread1, run1 = create_thread_and_run(
    "I need to solve the equation `3x + 11 = 14`. Can you help me?"
)
thread2, run2 = create_thread_and_run("Could you explain linear algebra to me?")
thread3, run3 = create_thread_and_run("I don't like math. What can I do?")

# Now all Runs are executing...
```

Once all Runs are going, we can wait on each and get the responses.


```python
import time

# Pretty printing helper
def pretty_print(messages):
    print("# Messages")
    for m in messages:
        print(f"{m.role}: {m.content[0].text.value}")
    print()


# Waiting in a loop
def wait_on_run(run, thread):
    while run.status == "queued" or run.status == "in_progress":
        run = client.beta.threads.runs.retrieve(
            thread_id=thread.id,
            run_id=run.id,
        )
        time.sleep(0.5)
    return run


# Wait for Run 1
run1 = wait_on_run(run1, thread1)
pretty_print(get_response(thread1))

# Wait for Run 2
run2 = wait_on_run(run2, thread2)
pretty_print(get_response(thread2))

# Wait for Run 3
run3 = wait_on_run(run3, thread3)
pretty_print(get_response(thread3))

# Thank our assistant on Thread 3 :)
run4 = submit_message(MATH_ASSISTANT_ID, thread3, "Thank you!")
run4 = wait_on_run(run4, thread3)
pretty_print(get_response(thread3))
```

```text
# Messages
user: I need to solve the equation `3x + 11 = 14`. Can you help me?
assistant: Sure! Subtract 11 from both sides to get \(3x = 3\), then divide by 3 to find \(x = 1\).

# Messages
user: Could you explain linear algebra to me?
assistant: Linear algebra is the branch of mathematics concerning vector spaces, linear transformations, and systems of linear equations, often represented with matrices.

# Messages
user: I don't like math. What can I do?
assistant: Try relating math to real-life interests or hobbies, practice with fun games or apps, and gradually build confidence with easier problems.

# Messages
user: I don't like math. What can I do?
assistant: Try relating math to real-life interests or hobbies, practice with fun games or apps, and gradually build confidence with easier problems.
user: Thank you!
assistant: You're welcome! If you have any more questions, feel free to ask!
```

Et voilà!

You may have noticed that this code is not actually specific to our math Assistant at all... this code will work for any new Assistant you create simply by changing the Assistant ID! That is the power of the Assistants API.


## Tools

A key feature of the Assistants API is the ability to equip our Assistants with Tools, like Code Interpreter, File Search, and custom Functions. Let's take a look at each.

### Code Interpreter

Let's equip our Math Tutor with the [Code Interpreter](https://platform.openai.com/docs/assistants/tools/code-interpreter) tool, which we can do from the Dashboard...


![Enabling code interpreter](https://developers.openai.com/cookbook/assets/images/assistants_overview_enable_code_interpreter.png)


...or the API, using the Assistant ID.


```python
assistant = client.beta.assistants.update(
    MATH_ASSISTANT_ID,
    tools=[{"type": "code_interpreter"}],
)
show_json(assistant)
```

```text
{'id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'created_at': 1736340398,
 'description': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'metadata': {},
 'model': 'gpt-4o',
 'name': 'Math Tutor',
 'object': 'assistant',
 'tools': [{'type': 'code_interpreter'}],
 'response_format': 'auto',
 'temperature': 1.0,
 'tool_resources': {'code_interpreter': {'file_ids': []}, 'file_search': None},
 'top_p': 1.0} 'tools': [{'type': 'code_interpreter'}],
 'response_format': 'auto',
 'temperature': 1.0,
 'tool_resources': {'code_interpreter': {'file_ids': []}, 'file_search': None},
 'top_p': 1.0}
```

Now, let's ask the Assistant to use its new tool.


```python
thread, run = create_thread_and_run(
    "Generate the first 20 fibbonaci numbers with code."
)
run = wait_on_run(run, thread)
pretty_print(get_response(thread))
```

```text
# Messages
user: Generate the first 20 fibbonaci numbers with code.
assistant: The first 20 Fibonacci numbers are: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987, 1597, 2584, 4181.
```

And that's it! The Assistant used Code Interpreter in the background, and gave us a final response.

For some use cases this may be enough – however, if we want more details on what precisely an Assistant is doing we can take a look at a Run's Steps.

### Steps


A Run is composed of one or more Steps. Like a Run, each Step has a `status` that you can query. This is useful for surfacing the progress of a Step to a user (e.g. a spinner while the Assistant is writing code or performing retrieval).


```python
run_steps = client.beta.threads.runs.steps.list(
    thread_id=thread.id, run_id=run.id, order="asc"
)
```

Let's take a look at each Step's `step_details`.


```python
for step in run_steps.data:
    step_details = step.step_details
    print(json.dumps(show_json(step_details), indent=4))
```

```text
{'tool_calls': [{'id': 'call_E1EE1loDmcWoc7FpkOMKYj6n',
   'code_interpreter': {'input': 'def generate_fibonacci(n):\n    fib_sequence = [0, 1]\n    while len(fib_sequence) < n:\n        next_value = fib_sequence[-1] + fib_sequence[-2]\n        fib_sequence.append(next_value)\n    return fib_sequence\n\n# Generate the first 20 Fibonacci numbers\nfirst_20_fibonacci = generate_fibonacci(20)\nfirst_20_fibonacci',
    'outputs': []},
   'type': 'code_interpreter'}],
 'type': 'tool_calls'}
```

```text
null
```

```text
{'message_creation': {'message_id': 'msg_RzTnbBMmzDYHk79a0x9qM5uU'},
 'type': 'message_creation'}
```

```text
null
```

We can see the `step_details` for two Steps:

1. `tool_calls` (plural, since it could be more than one in a single Step)
2. `message_creation`

The first Step is a `tool_calls`, specifically using the `code_interpreter` which contains:

- `input`, which was the Python code generated before the tool was called, and
- `output`, which was the result of running the Code Interpreter.

The second Step is a `message_creation`, which contains the `message` that was added to the Thread to communicate the results to the user.


### File search

Another powerful tool in the Assistants API is [File search](https://platform.openai.com/docs/assistants/tools/file-search). This allows the uploading of files to the Assistant to be used as a knowledge base when answering questions.


![Enabling retrieval](https://developers.openai.com/cookbook/assets/images/assistants_overview_enable_retrieval.png)


```python
# Upload the file
file = client.files.create(
    file=open(
        "data/language_models_are_unsupervised_multitask_learners.pdf",
        "rb",
    ),
    purpose="assistants",
)

# Create a vector store
vector_store = client.beta.vector_stores.create(
    name="language_models_are_unsupervised_multitask_learners",
)

# Add the file to the vector store
vector_store_file = client.beta.vector_stores.files.create_and_poll(
    vector_store_id=vector_store.id,
    file_id=file.id,
)

# Confirm the file was added
while vector_store_file.status == "in_progress":
    time.sleep(1)
if vector_store_file.status == "completed":
    print("File added to vector store")
elif vector_store_file.status == "failed":
    raise Exception("Failed to add file to vector store")

# Update Assistant
assistant = client.beta.assistants.update(
    MATH_ASSISTANT_ID,
    tools=[{"type": "code_interpreter"}, {"type": "file_search"}],
    tool_resources={
        "file_search":{
            "vector_store_ids": [vector_store.id]
        },
        "code_interpreter": {
            "file_ids": [file.id]
        }
    },
)
show_json(assistant)
```

```text
File added to vector store
```

```text
{'id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'created_at': 1736340398,
 'description': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'metadata': {},
 'model': 'gpt-4o',
 'name': 'Math Tutor',
 'object': 'assistant',
 'tools': [{'type': 'code_interpreter'},
  {'type': 'file_search',
   'file_search': {'max_num_results': None,
    'ranking_options': {'score_threshold': 0.0,
     'ranker': 'default_2024_08_21'}}}],
 'response_format': 'auto',
 'temperature': 1.0,
 'tool_resources': {'code_interpreter': {'file_ids': ['file-GQFm2i7N8LrAQatefWKEsE']},
  'file_search': {'vector_store_ids': ['vs_dEArILZSJh7J799QACi3QhuU']}},
 'top_p': 1.0}
```

```python
thread, run = create_thread_and_run(
    "What are some cool math concepts behind this ML paper pdf? Explain in two sentences."
)
run = wait_on_run(run, thread)
pretty_print(get_response(thread))
```

```text
# Messages
user: What are some cool math concepts behind this ML paper pdf? Explain in two sentences.
assistant: The paper explores the concept of multitask learning where a single model is used to perform various tasks, modeling the conditional distribution \( p(\text{output} | \text{input, task}) \), inspired by probabilistic approaches【6:10†source】. It also discusses the use of Transformer-based architectures and parallel corpus substitution in language models, enhancing their ability to generalize across domain tasks without explicit task-specific supervision【6:2†source】【6:5†source】.
```

> **Note**
> There are more intricacies in File Search, like [Annotations](https://platform.openai.com/docs/assistants/how-it-works/managing-threads-and-messages), which may be covered in another cookbook.


```python
# Delete the vector store
client.beta.vector_stores.delete(vector_store.id)
```

```text
VectorStoreDeleted(id='vs_dEArILZSJh7J799QACi3QhuU', deleted=True, object='vector_store.deleted')
```

### Functions

As a final powerful tool for your Assistant, you can specify custom [Functions](https://platform.openai.com/docs/assistants/tools/function-calling) (much like the [Function Calling](https://platform.openai.com/docs/guides/function-calling) in the Chat Completions API). During a Run, the Assistant can then indicate it wants to call one or more functions you specified. You are then responsible for calling the Function, and providing the output back to the Assistant.

Let's take a look at an example by defining a `display_quiz()` Function for our Math Tutor.

This function will take a `title` and an array of `question`s, display the quiz, and get input from the user for each:

- `title`
- `questions`
  - `question_text`
  - `question_type`: [`MULTIPLE_CHOICE`, `FREE_RESPONSE`]
  - `choices`: ["choice 1", "choice 2", ...]

I'll mocking out responses with `get_mock_response...`. This is where you'd get the user's actual input.


```python
def get_mock_response_from_user_multiple_choice():
    return "a"


def get_mock_response_from_user_free_response():
    return "I don't know."


def display_quiz(title, questions):
    print("Quiz:", title)
    print()
    responses = []

    for q in questions:
        print(q["question_text"])
        response = ""

        # If multiple choice, print options
        if q["question_type"] == "MULTIPLE_CHOICE":
            for i, choice in enumerate(q["choices"]):
                print(f"{i}. {choice}")
            response = get_mock_response_from_user_multiple_choice()

        # Otherwise, just get response
        elif q["question_type"] == "FREE_RESPONSE":
            response = get_mock_response_from_user_free_response()

        responses.append(response)
        print()

    return responses
```

Here's what a sample quiz would look like:


```python
responses = display_quiz(
    "Sample Quiz",
    [
        {"question_text": "What is your name?", "question_type": "FREE_RESPONSE"},
        {
            "question_text": "What is your favorite color?",
            "question_type": "MULTIPLE_CHOICE",
            "choices": ["Red", "Blue", "Green", "Yellow"],
        },
    ],
)
print("Responses:", responses)
```

```text
Quiz: Sample Quiz

What is your name?

What is your favorite color?
0. Red
1. Blue
2. Green
3. Yellow

Responses: ["I don't know.", 'a']
```

Now, let's define the interface of this function in JSON format, so our Assistant can call it:


```python
function_json = {
    "name": "display_quiz",
    "description": "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    "parameters": {
        "type": "object",
        "properties": {
            "title": {"type": "string"},
            "questions": {
                "type": "array",
                "description": "An array of questions, each with a title and potentially options (if multiple choice).",
                "items": {
                    "type": "object",
                    "properties": {
                        "question_text": {"type": "string"},
                        "question_type": {
                            "type": "string",
                            "enum": ["MULTIPLE_CHOICE", "FREE_RESPONSE"]
                        },
                        "choices": {"type": "array", "items": {"type": "string"}}
                    },
                    "required": ["question_text"]
                }
            }
        },
        "required": ["title", "questions"]
    }
}
```

Once again, let's update our Assistant either through the Dashboard or the API.


![Enabling custom function](https://developers.openai.com/cookbook/assets/images/assistants_overview_enable_function.png)

> **Note**
> Pasting the function JSON into the Dashboard was a bit finicky due to indentation, etc. I just asked ChatGPT to format my function the same as one of the examples on the Dashboard :).


```python
assistant = client.beta.assistants.update(
    MATH_ASSISTANT_ID,
    tools=[
        {"type": "code_interpreter"},
        {"type": "file_search"},
        {"type": "function", "function": function_json},
    ],
)
show_json(assistant)
```

```text
{'id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'created_at': 1736340398,
 'description': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'metadata': {},
 'model': 'gpt-4o',
 'name': 'Math Tutor',
 'object': 'assistant',
 'tools': [{'type': 'code_interpreter'},
  {'type': 'file_search',
   'file_search': {'max_num_results': None,
    'ranking_options': {'score_threshold': 0.0,
     'ranker': 'default_2024_08_21'}}},
  {'function': {'name': 'display_quiz',
    'description': "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    'description': "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    'parameters': {'type': 'object',
     'properties': {'title': {'type': 'string'},
      'questions': {'type': 'array',
       'description': 'An array of questions, each with a title and potentially options (if multiple choice).',
       'items': {'type': 'object',
        'properties': {'question_text': {'type': 'string'},
         'question_type': {'type': 'string',
          'enum': ['MULTIPLE_CHOICE', 'FREE_RESPONSE']},
         'choices': {'type': 'array', 'items': {'type': 'string'}}},
        'required': ['question_text']}}},
     'required': ['title', 'questions']},
    'strict': False},
   'type': 'function'}],
 'response_format': 'auto',
 'temperature': 1.0,
 'tool_resources': {'code_interpreter': {'file_ids': ['file-GQFm2i7N8LrAQatefWKEsE']},
  'file_search': {'vector_store_ids': []}},
 'top_p': 1.0}
```

And now, we ask for a quiz.


```python
thread, run = create_thread_and_run(
    "Make a quiz with 2 questions: One open ended, one multiple choice. Then, give me feedback for the responses."
)
run = wait_on_run(run, thread)
run.status
```

```text
'requires_action'
```

Now, however, when we check the Run's `status` we see `requires_action`! Let's take a closer.


```python
show_json(run)
```

```text
{'id': 'run_ekMRSI2h35asEzKirRf4BTwZ',
 'assistant_id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'cancelled_at': None,
 'completed_at': None,
 'created_at': 1736341020,
 'expires_at': 1736341620,
 'failed_at': None,
 'incomplete_details': None,
 'incomplete_details': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'last_error': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'metadata': {},
 'model': 'gpt-4o',
 'object': 'thread.run',
 'parallel_tool_calls': True,
 'required_action': {'submit_tool_outputs': {'tool_calls': [{'id': 'call_uvJEn0fxM4sgmzek8wahBGLi',
     'function': {'arguments': '{"title":"Math Quiz","questions":[{"question_text":"What is the derivative of the function f(x) = 3x^2 + 2x - 5?","question_type":"FREE_RESPONSE"},{"question_text":"What is the value of \\\\( \\\\int_{0}^{1} 2x \\\\, dx \\\\)?","question_type":"MULTIPLE_CHOICE","choices":["0","1","2","3"]}]}',
      'name': 'display_quiz'},
     'type': 'function'}]},
  'type': 'submit_tool_outputs'},
 'response_format': 'auto',
 'started_at': 1736341022,
 'status': 'requires_action',
 'thread_id': 'thread_8bK2PXfoeijEHBVEzYuJXt17',
 'tool_choice': 'auto',
 'tools': [{'type': 'code_interpreter'},
  {'type': 'file_search',
   'file_search': {'max_num_results': None,
    'ranking_options': {'score_threshold': 0.0,
     'ranker': 'default_2024_08_21'}}},
  {'function': {'name': 'display_quiz',
    'description': "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    'description': "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    'parameters': {'type': 'object',
     'properties': {'title': {'type': 'string'},
      'questions': {'type': 'array',
       'description': 'An array of questions, each with a title and potentially options (if multiple choice).',
       'items': {'type': 'object',
        'properties': {'question_text': {'type': 'string'},
         'question_type': {'type': 'string',
          'enum': ['MULTIPLE_CHOICE', 'FREE_RESPONSE']},
         'choices': {'type': 'array', 'items': {'type': 'string'}}},
        'required': ['question_text']}}},
     'required': ['title', 'questions']},
    'strict': False},
   'type': 'function'}],
 'truncation_strategy': {'type': 'auto', 'last_messages': None},
 'usage': None,
 'temperature': 1.0,
 'top_p': 1.0,
 'tool_resources': {}}    'strict': False},
   'type': 'function'}],
 'truncation_strategy': {'type': 'auto', 'last_messages': None},
 'usage': None,
 'temperature': 1.0,
 'top_p': 1.0,
 'tool_resources': {}}
```

The `required_action` field indicates a Tool is waiting for us to run it and submit its output back to the Assistant. Specifically, the `display_quiz` function! Let's start by parsing the `name` and `arguments`.

> **Note**
> While in this case we know there is only one Tool call, in practice the Assistant may choose to call multiple tools.


```python
# Extract single tool call
tool_call = run.required_action.submit_tool_outputs.tool_calls[0]
name = tool_call.function.name
arguments = json.loads(tool_call.function.arguments)

print("Function Name:", name)
print("Function Arguments:")
arguments
```

```text
Function Name: display_quiz
Function Arguments:
```

```text
{'title': 'Math Quiz',
 'questions': [{'question_text': 'What is the derivative of the function f(x) = 3x^2 + 2x - 5?',
   'question_type': 'FREE_RESPONSE'},
  {'question_text': 'What is the value of \\( \\int_{0}^{1} 2x \\, dx \\)?',
   'question_type': 'MULTIPLE_CHOICE',
   'choices': ['0', '1', '2', '3']}]}
```

Now let's actually call our `display_quiz` function with the arguments provided by the Assistant:


```python
responses = display_quiz(arguments["title"], arguments["questions"])
print("Responses:", responses)
```

```text
Quiz: Math Quiz
Quiz: Math Quiz

What is the derivative of the function f(x) = 3x^2 + 2x - 5?

What is the value of \( \int_{0}^{1} 2x \, dx \)?
0. 0
1. 1
2. 2
3. 3

Responses: ["I don't know.", 'a']
```

Great! (Remember these responses are the one's we mocked earlier. In reality, we'd be getting input from the back from this function call.)

Now that we have our responses, let's submit them back to the Assistant. We'll need the `tool_call` ID, found in the `tool_call` we parsed out earlier. We'll also need to encode our `list`of responses into a `str`.


```python
run = client.beta.threads.runs.submit_tool_outputs(
    thread_id=thread.id,
    run_id=run.id,
    tool_outputs=tool_outputs
)
show_json(run)
```

```text
{'id': 'run_ekMRSI2h35asEzKirRf4BTwZ',
 'assistant_id': 'asst_qvXmYlZV8zhABI2RtPzDfV6z',
 'cancelled_at': None,
 'completed_at': None,
 'created_at': 1736341020,
 'expires_at': 1736341620,
 'failed_at': None,
 'incomplete_details': None,
 'incomplete_details': None,
 'instructions': 'You are a personal math tutor. Answer questions briefly, in a sentence or less.',
 'last_error': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'max_completion_tokens': None,
 'max_prompt_tokens': None,
 'metadata': {},
 'model': 'gpt-4o',
 'object': 'thread.run',
 'parallel_tool_calls': True,
 'parallel_tool_calls': True,
 'required_action': None,
 'response_format': 'auto',
 'started_at': 1736341022,
 'status': 'queued',
 'thread_id': 'thread_8bK2PXfoeijEHBVEzYuJXt17',
 'tool_choice': 'auto',
 'tools': [{'type': 'code_interpreter'},
  {'type': 'file_search',
   'file_search': {'max_num_results': None,
    'ranking_options': {'score_threshold': 0.0,
     'ranker': 'default_2024_08_21'}}},
  {'function': {'name': 'display_quiz',
    'description': "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    'description': "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
    'parameters': {'type': 'object',
     'properties': {'title': {'type': 'string'},
      'questions': {'type': 'array',
       'description': 'An array of questions, each with a title and potentially options (if multiple choice).',
       'items': {'type': 'object',
        'properties': {'question_text': {'type': 'string'},
         'question_type': {'type': 'string',
          'enum': ['MULTIPLE_CHOICE', 'FREE_RESPONSE']},
         'choices': {'type': 'array', 'items': {'type': 'string'}}},
        'required': ['question_text']}}},
     'required': ['title', 'questions']},
    'strict': False},
   'type': 'function'}],
 'truncation_strategy': {'type': 'auto', 'last_messages': None},
 'usage': None,
 'temperature': 1.0,
 'top_p': 1.0,
 'tool_resources': {}}    'strict': False},
   'type': 'function'}],
 'truncation_strategy': {'type': 'auto', 'last_messages': None},
 'usage': None,
 'temperature': 1.0,
 'top_p': 1.0,
 'tool_resources': {}}
```

We can now wait for the Run to complete once again, and check our Thread!


```python
run = wait_on_run(run, thread)
pretty_print(get_response(thread))
```

```text
# Messages
user: Make a quiz with 2 questions: One open ended, one multiple choice. Then, give me feedback for the responses.
assistant: Since no specific information was found in the uploaded file, I'll create a general math quiz for you:

1. **Open-ended Question**: What is the derivative of the function \( f(x) = 3x^2 + 2x - 5 \)?

2. **Multiple Choice Question**: What is the value of \( \int_{0}^{1} 2x \, dx \)?
    - A) 0
    - B) 1
    - C) 2
    - D) 3

I will now present the quiz to you for response.
assistant: Here is the feedback for your responses:

1. **Derivative Question**: 
   - Your Response: "I don't know."
   - Feedback: The derivative of \( f(x) = 3x^2 + 2x - 5 \) is \( f'(x) = 6x + 2 \).

2. **Integration Question**: 
   - Your Response: A) 0
   - Feedback: The correct answer is B) 1. The integration \(\int_{0}^{1} 2x \, dx \) evaluates to 1.
```

Woohoo 🎉


## Conclusion

We covered a lot of ground in this notebook, give yourself a high-five! Hopefully you should now have a strong foundation to build powerful, stateful experiences with tools like Code Interpreter, Retrieval, and Functions!

There's a few sections we didn't cover for the sake of brevity, so here's a few resources to explore further:

- [Annotations](https://platform.openai.com/docs/assistants/how-it-works/managing-threads-and-messages): parsing file citations
- [Files](https://platform.openai.com/docs/api-reference/assistants/file-object): Thread scoped vs Assistant scoped
- [Parallel Function Calls](https://platform.openai.com/docs/guides/function-calling/parallel-function-calling): calling multiple tools in a single Step
- Multi-Assistant Thread Runs: single Thread with Messages from multiple Assistants
- Streaming: coming soon!

Now go off and build something ama[zing](https://www.youtube.com/watch?v=xvFZjo5PgG0&pp=ygUQcmljayByb2xsIG5vIGFkcw%3D%3D)!