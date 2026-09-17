# Unit test writing using a multi-step prompt (with the older API)

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

Complex tasks, such as writing unit tests, can benefit from multi-step prompts. In contrast to a single prompt, a multi-step prompt generates text from GPT-3 and then feeds that text back into subsequent prompts. This can help in cases where you want GPT-3 to explain its reasoning before answering, or brainstorm a plan before executing it.

In this notebook, we use a 3-step prompt to write unit tests in Python using the following steps:

1. Given a Python function, we first prompt GPT-3 to explain what the function is doing.
2. Second, we prompt GPT-3 to plan a set of unit tests for the function.
    - If the plan is too short, we ask GPT-3 to elaborate with more ideas for unit tests.
3. Finally, we prompt GPT-3 to write the unit tests.

The code example illustrates a few optional embellishments on the chained, multi-step prompt:

- Conditional branching (e.g., only asking for elaboration if the first plan is too short)
- Different models for different steps (e.g., `gpt-3.5-turbo-instruct` for the text planning steps and `gpt-4` for the code writing step)
- A check that re-runs the function if the output is unsatisfactory (e.g., if the output code cannot be parsed by Python's `ast` module)
- Streaming output so that you can start reading the output before it's fully generated (useful for long, multi-step outputs)

The full 3-step prompt looks like this (using as an example `pytest` for the unit test framework and `is_palindrome` as the function):

    # How to write great unit tests with pytest

    In this advanced tutorial for experts, we'll use Python 3.9 and `pytest` to write a suite of unit tests to verify the behavior of the following function.
    ```python
    def is_palindrome(s):
        return s == s[::-1]
    ```

    Before writing any unit tests, let's review what each element of the function is doing exactly and what the author's intentions may have been.
    - First,{GENERATED IN STEP 1}
        
    A good unit test suite should aim to:
    - Test the function's behavior for a wide range of possible inputs
    - Test edge cases that the author may not have foreseen
    - Take advantage of the features of `pytest` to make the tests easy to write and maintain
    - Be easy to read and understand, with clean code and descriptive names
    - Be deterministic, so that the tests always pass or fail in the same way

    `pytest` has many convenient features that make it easy to write and maintain unit tests. We'll use them to write unit tests for the function above.

    For this particular function, we'll want our unit tests to handle the following diverse scenarios (and under each scenario, we include a few examples as sub-bullets):
    -{GENERATED IN STEP 2}

    [OPTIONALLY APPENDED]In addition to the scenarios above, we'll also want to make sure we don't forget to test rare or unexpected edge cases (and under each edge case, we include a few examples as sub-bullets):
    -{GENERATED IN STEP 2B}

    Before going into the individual tests, let's first look at the complete suite of unit tests as a cohesive whole. We've added helpful comments to explain what each line does.
    ```python
    import pytest  # used for our unit tests

    def is_palindrome(s):
        return s == s[::-1]

    #Below, each test case is represented by a tuple passed to the @pytest.mark.parametrize decorator
    {GENERATED IN STEP 3}

````python
import ast  # used for detecting whether generated Python code is valid
import openai

# example of a function that uses a multi-step prompt to write unit tests
def unit_test_from_function(
    function_to_test: str,  # Python function to test, as a string
    unit_test_package: str = "pytest",  # unit testing package; use the name as it appears in the import statement
    approx_min_cases_to_cover: int = 7,  # minimum number of test case categories to cover (approximate)
    print_text: bool = False,  # optionally prints text; helpful for understanding the function & debugging
    text_model: str = "gpt-3.5-turbo-instruct",  # model used to generate text plans in steps 1, 2, and 2b
    code_model: str = "gpt-3.5-turbo-instruct",  # if you don't have access to code models, you can use text models here instead
    max_tokens: int = 1000,  # can set this high, as generations should be stopped earlier by stop sequences
    temperature: float = 0.4,  # temperature = 0 can sometimes get stuck in repetitive loops, so we use 0.4
    reruns_if_fail: int = 1,  # if the output code cannot be parsed, this will re-run the function up to N times
) -> str:
    """Outputs a unit test for a given Python function, using a 3-step GPT-3 prompt."""

    # Step 1: Generate an explanation of the function

    # create a markdown-formatted prompt that asks GPT-3 to complete an explanation of the function, formatted as a bullet list
    prompt_to_explain_the_function = f"""# How to write great unit tests with {unit_test_package}

In this advanced tutorial for experts, we'll use Python 3.9 and `{unit_test_package}` to write a suite of unit tests to verify the behavior of the following function.
```python
{function_to_test}
```

Before writing any unit tests, let's review what each element of the function is doing exactly and what the author's intentions may have been.
- First,"""
    if print_text:
        text_color_prefix = "\033[30m"  # black; if you read against a dark background \033[97m is white
        print(text_color_prefix + prompt_to_explain_the_function, end="")  # end='' prevents a newline from being printed

    # send the prompt to the API, using \n\n as a stop sequence to stop at the end of the bullet list
    explanation_response = openai.Completion.create(
        model=text_model,
        prompt=prompt_to_explain_the_function,
        stop=["\n\n", "\n\t\n", "\n    \n"],
        max_tokens=max_tokens,
        temperature=temperature,
        stream=True,
    )
    explanation_completion = ""
    if print_text:
        completion_color_prefix = "\033[92m"  # green
        print(completion_color_prefix, end="")
    for event in explanation_response:
        event_text = event["choices"][0]["text"]
        explanation_completion += event_text
        if print_text:
            print(event_text, end="")

    # Step 2: Generate a plan to write a unit test

    # create a markdown-formatted prompt that asks GPT-3 to complete a plan for writing unit tests, formatted as a bullet list
    prompt_to_explain_a_plan = f"""
    
A good unit test suite should aim to:
- Test the function's behavior for a wide range of possible inputs
- Test edge cases that the author may not have foreseen
- Take advantage of the features of `{unit_test_package}` to make the tests easy to write and maintain
- Be easy to read and understand, with clean code and descriptive names
- Be deterministic, so that the tests always pass or fail in the same way

`{unit_test_package}` has many convenient features that make it easy to write and maintain unit tests. We'll use them to write unit tests for the function above.

For this particular function, we'll want our unit tests to handle the following diverse scenarios (and under each scenario, we include a few examples as sub-bullets):
-"""
    if print_text:
        print(text_color_prefix + prompt_to_explain_a_plan, end="")

    # append this planning prompt to the results from step 1
    prior_text = prompt_to_explain_the_function + explanation_completion
    full_plan_prompt = prior_text + prompt_to_explain_a_plan

    # send the prompt to the API, using \n\n as a stop sequence to stop at the end of the bullet list
    plan_response = openai.Completion.create(
        model=text_model,
        prompt=full_plan_prompt,
        stop=["\n\n", "\n\t\n", "\n    \n"],
        max_tokens=max_tokens,
        temperature=temperature,
        stream=True,
    )
    plan_completion = ""
    if print_text:
        print(completion_color_prefix, end="")
    for event in plan_response:
        event_text = event["choices"][0]["text"]
        plan_completion += event_text
        if print_text:
            print(event_text, end="")

    # Step 2b: If the plan is short, ask GPT-3 to elaborate further
    # this counts top-level bullets (e.g., categories), but not sub-bullets (e.g., test cases)
    elaboration_needed = plan_completion.count("\n-") +1 < approx_min_cases_to_cover  # adds 1 because the first bullet is not counted
    if elaboration_needed:
        prompt_to_elaborate_on_the_plan = f"""

In addition to the scenarios above, we'll also want to make sure we don't forget to test rare or unexpected edge cases (and under each edge case, we include a few examples as sub-bullets):
-"""
        if print_text:
            print(text_color_prefix + prompt_to_elaborate_on_the_plan, end="")

        # append this elaboration prompt to the results from step 2
        prior_text = full_plan_prompt + plan_completion
        full_elaboration_prompt = prior_text + prompt_to_elaborate_on_the_plan

        # send the prompt to the API, using \n\n as a stop sequence to stop at the end of the bullet list
        elaboration_response = openai.Completion.create(
            model=text_model,
            prompt=full_elaboration_prompt,
            stop=["\n\n", "\n\t\n", "\n    \n"],
            max_tokens=max_tokens,
            temperature=temperature,
            stream=True,
        )
        elaboration_completion = ""
        if print_text:
            print(completion_color_prefix, end="")
        for event in elaboration_response:
            event_text = event["choices"][0]["text"]
            elaboration_completion += event_text
            if print_text:
                print(event_text, end="")

    # Step 3: Generate the unit test

    # create a markdown-formatted prompt that asks GPT-3 to complete a unit test
    starter_comment = ""
    if unit_test_package == "pytest":
        starter_comment = "Below, each test case is represented by a tuple passed to the @pytest.mark.parametrize decorator"
    prompt_to_generate_the_unit_test = f"""

Before going into the individual tests, let's first look at the complete suite of unit tests as a cohesive whole. We've added helpful comments to explain what each line does.
```python
import {unit_test_package}  # used for our unit tests

{function_to_test}

#{starter_comment}"""
    if print_text:
        print(text_color_prefix + prompt_to_generate_the_unit_test, end="")

    # append this unit test prompt to the results from step 3
    if elaboration_needed:
        prior_text = full_elaboration_prompt + elaboration_completion
    else:
        prior_text = full_plan_prompt + plan_completion
    full_unit_test_prompt = prior_text + prompt_to_generate_the_unit_test

    # send the prompt to the API, using ``` as a stop sequence to stop at the end of the code block
    unit_test_response = openai.Completion.create(
        model=code_model,
        prompt=full_unit_test_prompt,
        stop="```",
        max_tokens=max_tokens,
        temperature=temperature,
        stream=True
    )
    unit_test_completion = ""
    if print_text:
        print(completion_color_prefix, end="")
    for event in unit_test_response:
        event_text = event["choices"][0]["text"]
        unit_test_completion += event_text
        if print_text:
            print(event_text, end="")

    # check the output for errors
    code_start_index = prompt_to_generate_the_unit_test.find("```python\n") + len("```python\n")
    code_output = prompt_to_generate_the_unit_test[code_start_index:] + unit_test_completion
    try:
        ast.parse(code_output)
    except SyntaxError as e:
        print(f"Syntax error in generated code: {e}")
        if reruns_if_fail > 0:
            print("Rerunning...")
            return unit_test_from_function(
                function_to_test=function_to_test,
                unit_test_package=unit_test_package,
                approx_min_cases_to_cover=approx_min_cases_to_cover,
                print_text=print_text,
                text_model=text_model,
                code_model=code_model,
                max_tokens=max_tokens,
                temperature=temperature,
                reruns_if_fail=reruns_if_fail-1,  # decrement rerun counter when calling again
            )

    # return the unit test as a string
    return unit_test_completion
````

```python
example_function = """def is_palindrome(s):
    return s == s[::-1]"""

unit_test_from_function(example_function, print_text=True)
```

````text
[30m# How to write great unit tests with pytest

In this advanced tutorial for experts, we'll use Python 3.9 and `pytest` to write a suite of unit tests to verify the behavior of the following function.
```python
def is_palindrome(s):
    return s == s[::-1]
```

Before writing any unit tests, let's review what each element of the function is doing exactly and what the author's intentions may have been.
- First,[92m we have a function definition. This is where we give the function a name, `is_palindrome`, and specify the arguments that the function accepts. In this case, the function accepts a single string argument, `s`.
- Next, we have a return statement. This is where we specify the value that the function returns. In this case, the function returns `s == s[::-1]`.
- Finally, we have a function call. This is where we actually call the function with a specific set of arguments. In this case, we're calling the function with the string `"racecar"`.[30m
    
A good unit test suite should aim to:
- Test the function's behavior for a wide range of possible inputs
- Test edge cases that the author may not have foreseen
- Take advantage of the features of `pytest` to make the tests easy to write and maintain
- Be easy to read and understand, with clean code and descriptive names
- Be deterministic, so that the tests always pass or fail in the same way

`pytest` has many convenient features that make it easy to write and maintain unit tests. We'll use them to write unit tests for the function above.

For this particular function, we'll want our unit tests to handle the following diverse scenarios (and under each scenario, we include a few examples as sub-bullets):
-[92m The input is a palindrome
    - `"racecar"`
    - `"madam"`
    - `"anna"`
- The input is not a palindrome
    - `"python"`
    - `"test"`
    - `"1234"`
- The input is an empty string
    - `""`
- The input is `None`
- The input is not a string
    - `1`
    - `1.0`
    - `True`
    - `False`
    - `[]`
    - `{}`[30m

In addition to the scenarios above, we'll also want to make sure we don't forget to test rare or unexpected edge cases (and under each edge case, we include a few examples as sub-bullets):
-[92m The input is a palindrome with spaces
    - `"race car"`
    - `" madam "`
    - `" anna "`
- The input is not a palindrome with spaces
    - `" python "`
    - `" test "`
    - `" 1234 "`
- The input is a palindrome with punctuation
    - `"racecar!"`
    - `"Madam, I'm Adam."`
    - `"Anna's"`
- The input is not a palindrome with punctuation
    - `"python!"`
    - `"test."`
    - `"1234!"`
- The input is a palindrome with mixed case
    - `"Racecar"`
    - `"Madam"`
    - `"Anna"`
- The input is not a palindrome with mixed case
    - `"Python"`
    - `"Test"`
    - `"1234"`[30m

Before going into the individual tests, let's first look at the complete suite of unit tests as a cohesive whole. We've added helpful comments to explain what each line does.
```python
import pytest  # used for our unit tests

def is_palindrome(s):
    return s == s[::-1]

#Below, each test case is represented by a tuple passed to the @pytest.mark.parametrize decorator[92m.
#The first element of the tuple is a name for the test case, and the second element is a list of arguments for the test case.
#The @pytest.mark.parametrize decorator will generate a separate test function for each test case.
#The generated test function will be named test_is_palindrome_<name> where <name> is the name of the test case.
#The generated test function will be given the arguments specified in the list of arguments for the test case.
#The generated test function will be given the fixture specified in the decorator, in this case the function itself.
#The generated test function will call the function with the arguments and assert that the result is equal to the expected value.
@pytest.mark.parametrize(
    "name,args,expected",
    [
        # Test the function's behavior for a wide range of possible inputs
        ("palindrome", ["racecar"], True),
        ("palindrome", ["madam"], True),
        ("palindrome", ["anna"], True),
        ("non-palindrome", ["python"], False),
        ("non-palindrome", ["test"], False),
        ("non-palindrome", ["1234"], False),
        ("empty string", [""], True),
        ("None", [None], False),
        ("non-string", [1], False),
        ("non-string", [1.0], False),
        ("non-string", [True], False),
        ("non-string", [False], False),
        ("non-string", [[]], False),
        ("non-string", [{}], False),
        # Test edge cases that the author may not have foreseen
        ("palindrome with spaces", ["race car"], True),
        ("palindrome with spaces", [" madam "], True),
        ("palindrome with spaces", [" anna "], True),
        ("non-palindrome with spaces", [" python "], False),
        ("non-palindrome with spaces", [" test "], False),
        ("non-palindrome with spaces", [" 1234 "], False),
        ("palindrome with punctuation", ["racecar!"], True),
        ("palindrome with punctuation", ["Madam, I'm Adam."], True),
        ("palindrome with punctuation", ["Anna's"], True),
        ("non-palindrome with punctuation", ["python!"], False),
        ("non-palindrome with punctuation", ["test."], False),
        ("non-palindrome with punctuation", ["1234!"], False),
        ("palindrome with mixed case", ["Racecar"], True),
        ("palindrome with mixed case", ["Madam"], True),
        ("palindrome with mixed case", ["Anna"], True),
        ("non-palindrome with mixed case", ["Python"], False),
        ("non-palindrome with mixed case", ["Test"], False),
        ("non-palindrome with mixed case", ["1234"], False),
    ],
)
def test_is_palindrome(is_palindrome, args, expected):
    assert is_palindrome(*args) == expected
````

```text
'.\n#The first element of the tuple is a name for the test case, and the second element is a list of arguments for the test case.\n#The @pytest.mark.parametrize decorator will generate a separate test function for each test case.\n#The generated test function will be named test_is_palindrome_<name> where <name> is the name of the test case.\n#The generated test function will be given the arguments specified in the list of arguments for the test case.\n#The generated test function will be given the fixture specified in the decorator, in this case the function itself.\n#The generated test function will call the function with the arguments and assert that the result is equal to the expected value.\n@pytest.mark.parametrize(\n    "name,args,expected",\n    [\n        # Test the function\'s behavior for a wide range of possible inputs\n        ("palindrome", ["racecar"], True),\n        ("palindrome", ["madam"], True),\n        ("palindrome", ["anna"], True),\n        ("non-palindrome", ["python"], False),\n        ("non-palindrome", ["test"], False),\n        ("non-palindrome", ["1234"], False),\n        ("empty string", [""], True),\n        ("None", [None], False),\n        ("non-string", [1], False),\n        ("non-string", [1.0], False),\n        ("non-string", [True], False),\n        ("non-string", [False], False),\n        ("non-string", [[]], False),\n        ("non-string", [{}], False),\n        # Test edge cases that the author may not have foreseen\n        ("palindrome with spaces", ["race car"], True),\n        ("palindrome with spaces", [" madam "], True),\n        ("palindrome with spaces", [" anna "], True),\n        ("non-palindrome with spaces", [" python "], False),\n        ("non-palindrome with spaces", [" test "], False),\n        ("non-palindrome with spaces", [" 1234 "], False),\n        ("palindrome with punctuation", ["racecar!"], True),\n        ("palindrome with punctuation", ["Madam, I\'m Adam."], True),\n        ("palindrome with punctuation", ["Anna\'s"], True),\n        ("non-palindrome with punctuation", ["python!"], False),\n        ("non-palindrome with punctuation", ["test."], False),\n        ("non-palindrome with punctuation", ["1234!"], False),\n        ("palindrome with mixed case", ["Racecar"], True),\n        ("palindrome with mixed case", ["Madam"], True),\n        ("palindrome with mixed case", ["Anna"], True),\n        ("non-palindrome with mixed case", ["Python"], False),\n        ("non-palindrome with mixed case", ["Test"], False),\n        ("non-palindrome with mixed case", ["1234"], False),\n    ],\n)\ndef test_is_palindrome(is_palindrome, args, expected):\n    assert is_palindrome(*args) == expected\n'
```