# How to make your completions outputs reproducible with the new seed parameter

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

**TLDR**: Developers can now specify `seed` parameter in the Chat Completion request to receive (mostly) consistent outputs. To help you keep track of these changes, we expose the `system_fingerprint` field. If this value is different, you may see different outputs due to changes we've made on our systems. Please note that this feature is in beta and only currently supported for `gpt-4-1106-preview` and `gpt-3.5-turbo-1106`.

### Context

Reproducibility has always been a big request from user communities when using our APIs. For instance, when granted the capability of getting reproducible numerical result, users can unlock quite a bit of use cases that’s sensitive to numerical changes.

#### Model level features for consistent outputs

The Chat Completions and Completions APIs are non-deterministic by default (which means model outputs may differ from request to request), but now offer some control towards deterministic outputs using a few model level controls.

This can unlock consistent completions which enables full control on the model behaviors for anything built on top of the APIs, and quite useful for reproducing results and testing so you know get peace of mind from knowing exactly what you’d get.

#### Implementing consistent outputs

To receive _mostly_ deterministic outputs across API calls:

- Set the `seed` parameter to any integer of your choice, but use the same value across requests. For example, `12345`.
- Set all other parameters (prompt, temperature, top_p, etc.) to the same values across requests.
- In the response, check the `system_fingerprint` field. The system fingerprint is an identifier for the current combination of model weights, infrastructure, and other configuration options used by OpenAI servers to generate the completion. It changes whenever you change request parameters, or OpenAI updates numerical configuration of the infrastructure serving our models (which may happen a few times a year).

If the `seed`, request parameters, and `system_fingerprint` all match across your requests, then model outputs will mostly be identical. There is a small chance that responses differ even when request parameters and `system_fingerprint` match, due to the inherent non-determinism of our models.


### Model level controls for consistent outputs - `seed` and `system_fingerprint`

##### `seed`

If specified, our system will make a best effort to sample deterministically, such that repeated requests with the same seed and parameters should return the same result. Determinism is not guaranteed, and you should refer to the `system_fingerprint` response parameter to monitor changes in the backend.

##### `system_fingerprint`

This fingerprint represents the backend configuration that the model runs with. It can be used in conjunction with the seed request parameter to understand when backend changes have been made that might impact determinism.This is the indicator on whether users should expect "almost always the same result".


## Example: Generating a short excerpt with a fixed seed

In this example, we will demonstrate how to generate a short excerpt using a fixed seed. This can be particularly useful in scenarios where you need to generate consistent results for testing, debugging, or for applications that require consistent outputs.

### Python SDK

> **Note**
> Switch to latest version of the SDK (`1.3.3` at time of writing).

```python
!pip install --upgrade openai # Switch to the latest version of OpenAI (1.3.3 at time of writing)
```

```python
import openai
import asyncio
from IPython.display import display, HTML

from utils.embeddings_utils import (
    get_embedding,
    distances_from_embeddings
)

GPT_MODEL = "gpt-3.5-turbo-1106"
```

```python
async def get_chat_response(
    system_message: str, user_request: str, seed: int = None, temperature: float = 0.7
):
    try:
        messages = [
            {"role": "system", "content": system_message},
            {"role": "user", "content": user_request},
        ]

        response = openai.chat.completions.create(
            model=GPT_MODEL,
            messages=messages,
            seed=seed,
            max_tokens=200,
            temperature=temperature,
        )

        response_content = response.choices[0].message.content
        system_fingerprint = response.system_fingerprint
        prompt_tokens = response.usage.prompt_tokens
        completion_tokens = response.usage.total_tokens - response.usage.prompt_tokens

        table = f"""
        <table>
        <tr><th>Response</th><td>{response_content}</td></tr>
        <tr><th>System Fingerprint</th><td>{system_fingerprint}</td></tr>
        <tr><th>Number of prompt tokens</th><td>{prompt_tokens}</td></tr>
        <tr><th>Number of completion tokens</th><td>{completion_tokens}</td></tr>
        </table>
        """
        display(HTML(table))

        return response_content
    except Exception as e:
        print(f"An error occurred: {e}")
        return None

def calculate_average_distance(responses):
    """
    This function calculates the average distance between the embeddings of the responses.
    The distance between embeddings is a measure of how similar the responses are.
    """
    # Calculate embeddings for each response
    response_embeddings = [get_embedding(response) for response in responses]

    # Compute distances between the first response and the rest
    distances = distances_from_embeddings(response_embeddings[0], response_embeddings[1:])

    # Calculate the average distance
    average_distance = sum(distances) / len(distances)

    # Return the average distance
    return average_distance
```

First, let's try generating few different versions of a short excerpt about "a journey to Mars" without the `seed` parameter. This is the default behavior:

```python
topic = "a journey to Mars"
system_message = "You are a helpful assistant."
user_request = f"Generate a short excerpt of news about {topic}."

responses = []


async def get_response(i):
    print(f'Output {i + 1}\n{"-" * 10}')
    response = await get_chat_response(
        system_message=system_message, user_request=user_request
    )
    return response


responses = await asyncio.gather(*[get_response(i) for i in range(5)])
average_distance = calculate_average_distance(responses)
print(f"The average similarity between responses is: {average_distance}")
```

```text
Output 1
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Mars mission reaches critical stage as spacecraft successfully enters orbit around the red planet. The historic journey, which began over a year ago, has captured the world's attention as scientists and astronauts prepare to land on Mars for the first time. The mission is expected to provide valuable insights into the planet's geology, atmosphere, and potential for sustaining human life in the future."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>76</td></tr>
        </table>

```text
Output 2
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance rover successfully landed on Mars, marking a major milestone in the mission to explore the red planet. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples of rock and soil for future return to Earth. This historic achievement paves the way for further exploration and potential human missions to Mars in the near future."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>76</td></tr>
        </table>

```text
Output 3
----------
```

<table>
        <tr><th>Response</th><td>"SpaceX successfully launched the first manned mission to Mars yesterday, marking a historic milestone in space exploration. The crew of four astronauts will spend the next six months traveling to the red planet, where they will conduct groundbreaking research and experiments. This mission represents a significant step towards establishing a human presence on Mars and paves the way for future interplanetary travel."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>72</td></tr>
        </table>

```text
Output 4
----------
```

<table>
        <tr><th>Response</th><td>"NASA's latest Mars mission exceeds expectations as the Perseverance rover uncovers tantalizing clues about the Red Planet's past. Scientists are thrilled by the discovery of ancient riverbeds and sedimentary rocks, raising hopes of finding signs of past life on Mars. With this exciting progress, the dream of sending humans to Mars feels closer than ever before."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>72</td></tr>
        </table>

```text
Output 5
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance Rover Successfully Lands on Mars, Begins Exploration Mission

In a historic moment for space exploration, NASA's Perseverance rover has successfully landed on the surface of Mars. After a seven-month journey, the rover touched down in the Jezero Crater, a location scientists believe may have once held a lake and could potentially contain signs of ancient microbial life.

The rover's primary mission is to search for evidence of past life on Mars and collect rock and soil samples for future return to Earth. Equipped with advanced scientific instruments, including cameras, spectrometers, and a drill, Perseverance will begin its exploration of the Martian surface, providing valuable data and insights into the planet's geology and potential habitability.

This successful landing marks a significant milestone in humanity's quest to understand the red planet and paves the way for future manned missions to Mars. NASA's Perseverance rover is poised to unravel the mysteries of Mars and unlock new possibilities</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>200</td></tr>
        </table>

```text
The average similarity between responses is: 0.1136714512418833
```

Now, let's try to tun the same code with a constant `seed` of 123 and `temperature` of 0 and compare the responses and `system_fingerprint`.

```python
SEED = 123
responses = []


async def get_response(i):
    print(f'Output {i + 1}\n{"-" * 10}')
    response = await get_chat_response(
        system_message=system_message,
        seed=SEED,
        temperature=0,
        user_request=user_request,
    )
    return response


responses = await asyncio.gather(*[get_response(i) for i in range(5)])

average_distance = calculate_average_distance(responses)
print(f"The average distance between responses is: {average_distance}")
```

```text
Output 1
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance Rover Successfully Lands on Mars

In a historic achievement, NASA's Perseverance rover has successfully landed on the surface of Mars, marking a major milestone in the exploration of the red planet. The rover, which traveled over 293 million miles from Earth, is equipped with state-of-the-art instruments designed to search for signs of ancient microbial life and collect rock and soil samples for future return to Earth. This mission represents a significant step forward in our understanding of Mars and the potential for human exploration of the planet in the future."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>113</td></tr>
        </table>

```text
Output 2
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as scientists and engineers continue to push the boundaries of space travel and expand our understanding of the universe."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>81</td></tr>
        </table>

```text
Output 3
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as NASA continues to push the boundaries of space exploration."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>72</td></tr>
        </table>

```text
Output 4
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as scientists and engineers continue to push the boundaries of space travel and expand our understanding of the universe."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>81</td></tr>
        </table>

```text
Output 5
----------
```

<table>
        <tr><th>Response</th><td>"NASA's Perseverance rover successfully lands on Mars, marking a historic milestone in space exploration. The rover is equipped with advanced scientific instruments to search for signs of ancient microbial life and collect samples for future return to Earth. This mission paves the way for future human exploration of the red planet, as scientists and engineers continue to push the boundaries of space travel."</td></tr>
        <tr><th>System Fingerprint</th><td>fp_772e8125bb</td></tr>
        <tr><th>Number of prompt tokens</th><td>29</td></tr>
        <tr><th>Number of completion tokens</th><td>74</td></tr>
        </table>

```text
The average distance between responses is: 0.0449054397632461
```

As we can observe, the `seed` parameter allows us to generate much more consistent results.

## Conclusion

We demonstrated how to use a fixed integer `seed` to generate consistent outputs from our model. This is particularly useful in scenarios where reproducibility is important. However, it's important to note that while the `seed` ensures consistency, it does not guarantee the quality of the output. Note that when you want to use reproducible outputs, you need to set the `seed` to the same integer across Chat Completions calls. You should also match any other parameters like `temperature`, `max_tokens` etc. Further extension of reproducible outputs could be to use consistent `seed` when benchmarking/evaluating the performance of different prompts or models, to ensure that each version is evaluated under the same conditions, making the comparisons fair and the results reliable.