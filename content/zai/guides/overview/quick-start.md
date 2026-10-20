> ## Documentation Index
> Fetch the complete documentation index at: https://docs.z.ai/llms.txt
> Use this file to discover all available pages before exploring further.

# Quick Start

> [!NOTE]
>
> Tired of limits? GLM Coding Plan — monthly access to world-class models for coding and office tasks, compatible with 20+ top agent tools. All from just \$18/month.  [Try it now →](https://z.ai/subscribe?utm_campaign=Platform_Ops&_channel_track_key=DaprgHIc)

## Getting Started

**1. Get API Key**

* Access [Z.AI Open Platform](https://z.ai/model-api), Register or Login.
* Access [Billing Page](https://z.ai/manage-apikey/billing) to top up if needed.
* Create a Standard API Key in the [API Keys](https://z.ai/manage-apikey/apikey-list) management page.
* Copy your API Key for use.

  > **[Z.AI Open Platform](https://z.ai/model-api)**
  >
  > Access [Z.AI Open Platform](https://z.ai/model-api), Register or Login.

  > **[API Keys Management](https://z.ai/manage-apikey/apikey-list)**
  >
  > Create a Standard API Key in the [API Keys](https://z.ai/manage-apikey/apikey-list) management page.

**2. Choose Model**

> The platform offers multiple models, and you can select the appropriate model based on your needs. For detailed model introductions, please refer to the [Models & Agents](/guides/overview/overview).

> **[GLM-5.3](/guides/llm/glm-5.3)**
>
> Z.ai’s latest flagship model, delivering comprehensive advancements in software engineering and agent capabilities.

> **[GLM-5.3-FLASH](/guides/vlm/glm-5.3-flash)**
>
> Multimodal Coding model, specializing in visual programming.

**3. Choose the Calling Method**

Our platform provides various development approaches; you can select the best fit for your project needs and tech stack.

> **[HTTP API](/guides/develop/http/introduction)**
>
> Standard RESTful API, compatible with all programming languages.

> **[Z.AI Python SDK](/guides/develop/python/introduction)**
>
> Official Python SDK, featuring full type hints and async support.

> **[Z.AI Java SDK](/guides/develop/java/introduction)**
>
> Official Java SDK, designed for high concurrency and availability.

> **[OpenAI Python SDK](/guides/develop/openai/python)**
>
> OpenAI SDK Compatibility, quickly migrating from OpenAI.

> **[API Reference](/api-reference)**
>
> Complete API documentation with parameter descriptions.

**4. Make API Call**

After preparing your `API Key` and selecting a model, you can start making API calls. Here are examples using `curl`, `Python SDK`, and `Java SDK`:

> [!WARNING]
>
> When using GLM Coding Plan, please follow the [tutorial](/devpack/quick-start) to configure your dedicated endpoint.

**cURL**

```bash theme={null}
curl -X POST "https://api.z.ai/api/paas/v4/chat/completions" \
-H "Content-Type: application/json" \
-H "Accept-Language: en-US,en" \
-H "Authorization: Bearer YOUR_API_KEY" \
-d '{
    "model": "glm-5.3",
    "messages": [
        {
            "role": "system",
            "content": "You are a helpful AI assistant."
        },
        {
            "role": "user",
            "content": "Hello, please introduce yourself."
        }
    ]
}'
```

**Official Python SDK**

**Install SDK**

```bash theme={null}
# Install latest version
pip install zai-sdk

# Or specify version
pip install zai-sdk==0.2.3
```

**Verify Installation**

```python theme={null}
import zai
print(zai.__version__)
```

**Usage Example**

```python theme={null}
from zai import ZaiClient

# Initialize client
client = ZaiClient(api_key="YOUR_API_KEY")

# Create chat completion request
response = client.chat.completions.create(
    model="glm-5.3",
    messages=[
        {
            "role": "system",
            "content": "You are a helpful AI assistant."
        },
        {
            "role": "user",
            "content": "Hello, please introduce yourself."
        }
    ]
)

# Get response
print(response.choices[0].message.content)
```

**Official Java SDK**

**Install SDK**

**Maven**

```xml theme={null}
<dependency>
    <groupId>ai.z.openapi</groupId>
    <artifactId>zai-sdk</artifactId>
    <version>0.3.5</version>
</dependency>
```

**Gradle (Groovy)**

```groovy theme={null}
implementation 'ai.z.openapi:zai-sdk:0.3.5'
```

**Usage Example**

```java theme={null}
import ai.z.openapi.ZaiClient;
import ai.z.openapi.service.model.*;
import java.util.Arrays;

public class QuickStart {
    public static void main(String[] args) {
        // Initialize client
        ZaiClient client = ZaiClient.builder().ofZAI()
            .apiKey("YOUR_API_KEY")
            .build();

        // Create chat completion request
        ChatCompletionCreateParams request = ChatCompletionCreateParams.builder()
            .model("glm-5.3")
            .messages(Arrays.asList(
                ChatMessage.builder()
                    .role(ChatMessageRole.USER.value())
                    .content("Hello, who are you?")
                    .build()
            ))
            .stream(false)
            .build();

        // Send request
        ChatCompletionResponse response = client.chat().createChatCompletion(request);

        // Get response
        System.out.println(response.getData().getChoices().get(0).getMessage().getContent());
    }
}
```

**OpenAI Python SDK**

**Install SDK**

```bash theme={null}
# Install or upgrade to latest version
pip install --upgrade 'openai>=1.0'
```

**Verify Installation**

```python theme={null}
python -c "import openai; print(openai.__version__)"
```

**Usage Example**

```python theme={null}
from openai import OpenAI

client = OpenAI(
    api_key="your-Z.AI-api-key",
    base_url="https://api.z.ai/api/paas/v4/"
)

completion = client.chat.completions.create(
    model="glm-5.3",
    messages=[
        {"role": "system", "content": "You are a smart and creative novelist"},
        {"role": "user", "content": "Please write a short fairy tale story as a fairy tale master"}
    ]
)

print(completion.choices[0].message.content)
```

**OpenAI NodeJs SDK**

**Install SDK**

```bash theme={null}
# Install or upgrade to latest version
npm install openai

# Or using yarn
yarn add openai
```

**Usage Example**

```javascript theme={null}
import OpenAI from "openai";

const client = new OpenAI({
    apiKey: "your-Z.AI-api-key",
    baseURL: "https://api.z.ai/api/paas/v4/"
});

async function main() {
    const completion = await client.chat.completions.create({
        model: "glm-5.3",
        messages: [
            { role: "system", content: "You are a helpful AI assistant." },
            { role: "user", content: "Hello, please introduce yourself." }
        ]
    });

    console.log(completion.choices[0].message.content);
}

main();
```

**OpenAI Java SDK**

**Install SDK**

**Maven**

```xml theme={null}
<dependency>
    <groupId>com.openai</groupId>
    <artifactId>openai-java</artifactId>
    <version>2.20.1</version>
</dependency>
```

**Gradle (Groovy)**

```groovy theme={null}
implementation 'com.openai:openai-java:2.20.1'
```

**Usage Example**

```java theme={null}
import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.chat.completions.ChatCompletion;
import com.openai.models.chat.completions.ChatCompletionCreateParams;

public class QuickStart {
    public static void main(String[] args) {
        // Initialize client
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey("your-Z.AI-api-key")
            .baseUrl("https://api.z.ai/api/paas/v4/")
            .build();

        // Create chat completion request
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .addSystemMessage("You are a helpful AI assistant.")
            .addUserMessage("Hello, please introduce yourself.")
            .model("glm-5.3")
            .build();

        // Send request and get response
        ChatCompletion chatCompletion = client.chat().completions().create(params);
        Object response = chatCompletion.choices().get(0).message().content();

        System.out.println(response);
    }
}
```

### Get More

> **[API Reference](/api-reference)**
>
> Access API Reference.

> **[Python SDK](https://github.com/zai-org/z-ai-sdk-python)**
>
> Access Python SDK Github

> **[Java SDK](https://github.com/zai-org/z-ai-sdk-java)**
>
> Access Java SDK Github


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.