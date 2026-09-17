# Using chained calls for reasoning structured outputs

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

The initially released versions (September 2024) of [o1](https://openai.com/index/introducing-openai-o1-preview/) reasoning models have advanced capabilities but do not have [structured outputs](https://platform.openai.com/docs/guides/structured-outputs/examples) support. 

This means that requests with o1 don't have reliable type-safety and rely on the prompt itself to return a useful JSON. 

In this guide, we'll explore two methods to prompt o1 models, specifically `o1-preview`, to return a valid JSON format when using the OpenAI API.

# Prompting

The simplest way to return a JSON response using `o1-preview` is to explicitly prompt it. 

Let's run through an example of:
- Fetching a wikipedia page of companies
- Determining which could benefit the most from AI capabilities
- Returning them in a JSON format, which could then be ingested by other systems

```python
import requests
from openai import OpenAI

client = OpenAI()

def fetch_html(url):
    response = requests.get(url)
    if response.status_code == 200:
        return response.text
    else:
        return None

url = "https://en.wikipedia.org/wiki/List_of_largest_companies_in_the_United_States_by_revenue"
html_content = fetch_html(url)

json_format = """
{
    companies: [
        {
            \"company_name\": \"OpenAI\",
            \"page_link\": \"https://en.wikipedia.org/wiki/OpenAI\",
            \"reason\": \"OpenAI would benefit because they are an AI company...\"
        }
    ]
}
"""

o1_response = client.chat.completions.create(
    model="o1-preview",
    messages=[
        {
            "role": "user", 
            "content": f"""
You are a business analyst designed to understand how AI technology could be used across large corporations.

- Read the following html and return which companies would benefit from using AI technology: {html_content}.
- Rank these propects by opportunity by comparing them and show me the top 3. Return only as a JSON with the following format: {json_format}"
"""
        }
    ]
)

print(o1_response.choices[0].message.content)
```

```text
{
    "companies": [
        {
            "company_name": "Walmart",
            "page_link": "https://en.wikipedia.org/wiki/Walmart",
            "reason": "Walmart could benefit from AI technology by enhancing their supply chain management, optimizing inventory levels, improving customer service through AI-powered chatbots, and providing personalized shopping experiences. AI can help Walmart forecast demand more accurately, reduce operational costs, and increase overall efficiency."
        },
        {
            "company_name": "UnitedHealth Group",
            "page_link": "https://en.wikipedia.org/wiki/UnitedHealth_Group",
            "reason": "UnitedHealth Group could leverage AI technology to improve patient care through predictive analytics, personalize treatment plans, detect fraudulent claims, and streamline administrative processes. AI can assist in early disease detection, improve diagnostic accuracy, and enhance data analysis for better health outcomes."
        },
        {
            "company_name": "Ford Motor Company",
            "page_link": "https://en.wikipedia.org/wiki/Ford_Motor_Company",
            "reason": "Ford Motor Company could benefit from AI technology by advancing autonomous vehicle development, optimizing manufacturing processes with automation and robotics, implementing predictive maintenance, and enhancing the in-car experience with AI-driven features. AI can help Ford improve safety, reduce production costs, and innovate new transportation solutions."
        }
    ]
}
```

Note that the response is already quite good - it returns the JSON with the appropriate responses. However, it runs into the same pitfalls as existing use-cases of prompt-only JSON inference: 
- You must manually process this JSON into your type-safe structure
- Model refusals are not [explicitly returned from the API as a separate structure](https://platform.openai.com/docs/guides/structured-outputs/refusals)

# Structured Outputs

Let's now do this with [structured outputs](https://platform.openai.com/docs/guides/structured-outputs). To enable this functionality, we’ll link the `o1-preview` response with a follow-up request to `gpt-4o-mini`, which can effectively process the data returned from the initial o1-preview response.

```python
from pydantic import BaseModel
from devtools import pprint

class CompanyData(BaseModel):
    company_name: str
    page_link: str
    reason: str

class CompaniesData(BaseModel):
    companies: list[CompanyData]

o1_response = client.chat.completions.create(
    model="o1-preview",
    messages=[
        {
            "role": "user", 
            "content": f"""
You are a business analyst designed to understand how AI technology could be used across large corporations.

- Read the following html and return which companies would benefit from using AI technology: {html_content}.
- Rank these propects by opportunity by comparing them and show me the top 3. Return each with {CompanyData.__fields__.keys()}
"""
        }
    ]
)

o1_response_content = o1_response.choices[0].message.content

response = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "user", 
            "content": f"""
Given the following data, format it with the given response format: {o1_response_content}
"""
        }
    ],
    response_format=CompaniesData,
)

pprint(response.choices[0].message.parsed)
```

```text
CompaniesData(
    companies=[
        CompanyData(
            company_name='Walmart',
            page_link='https://en.wikipedia.org/wiki/Walmart',
            reason=(
                'As the largest retailer, Walmart can significantly benefit from AI by optimizing supply chain and inv'
                'entory management, improving demand forecasting, personalizing customer experiences, and enhancing in'
                '-store operations through AI-driven analytics.'
            ),
        ),
        CompanyData(
            company_name='JPMorgan Chase',
            page_link='https://en.wikipedia.org/wiki/JPMorgan_Chase',
            reason=(
                'As a leading financial institution, JPMorgan Chase can leverage AI for fraud detection, risk manageme'
                'nt, personalized banking services, algorithmic trading, and enhancing customer service with AI-powere'
                'd chatbots and virtual assistants.'
            ),
        ),
        CompanyData(
            company_name='UnitedHealth Group',
            page_link='https://en.wikipedia.org/wiki/UnitedHealth_Group',
            reason=(
                'Being a major player in healthcare, UnitedHealth Group can utilize AI to improve patient care through'
                ' predictive analytics, enhance diagnostics, streamline administrative processes, and reduce costs by '
                'optimizing operations with AI-driven solutions.'
            ),
        ),
    ],
)
```

# Conclusion

Structured outputs allow your code to have reliable type-safety and simpler prompting. In addition, it allows you to reuse your object schemas for easier integration into your existing workflows.

The o1 class of models currently doesn't have structured outputs support, but we can reuse existing structured outputs functionality from `gpt-4o-mini` by chaining two requests together. This flow currently requires two calls, but the second `gpt-4o-mini` call cost should be minimal compared to the `o1-preview`/`o1-mini` calls.