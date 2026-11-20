# Models and inference

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

Use the access token saved during [Sign in](https://developers.openai.com/siwc/token-sharing-open-source/sign-in) to populate a model picker and complete the first inference request.

## 1. List available models

Before showing a model picker, request the selected ChatGPT account’s model catalog with the same access token you will use for inference:

```bash
curl --fail-with-body --silent --show-error \
  https://api.openai.com/v1/models \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  | jq '[.models[] | select(.visibility == "list") | {slug, display_name}]'
```

The model-list response contains a `models` array. This example keeps models intended for display (`visibility: "list"`) and preserves the server's ordering. Show `display_name` in your UI and pass the selected `slug` as `model` in the next step. Refresh these choices when the user switches ChatGPT accounts. If you use Codex app-server, its `model/list` RPC may use a bundled or cached catalog. Use the request above when your UI needs current account-specific model choices.

## 2. Call the Responses API

To perform inference on behalf of the user, send the OAuth access token as `Authorization: Bearer <ACCESS_TOKEN>` to `POST https://api.openai.com/v1/responses`.

**Supported endpoint:** Use the public Responses API endpoint above for this ChatGPT plan usage flow; do not point it at ChatGPT's `backend-api` endpoints. Use a model available to the signed-in account.

**Set `store` to `false` and `stream` to `true` on each HTTP inference request in this flow.** Consume the response as a stream of events.

Read the saved `access_token` for the selected ChatGPT account into the process environment as `ACCESS_TOKEN` before running these examples.

### curl example

```bash
curl --no-buffer https://api.openai.com/v1/responses \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-6.1-sol",
    "input": [
      {
        "role": "user",
        "content": "Say exactly: Hello, world!"
      }
    ],
    "store": false,
    "stream": true
  }'
```




### SDK example

Set `ACCESS_TOKEN` to the OAuth access token. The SDK sends it as the bearer credential.




For TypeScript, install the SDK with `npm install openai`. Save the complete example below as `token-sharing.ts` in the directory where you installed the SDK, then run `node --experimental-strip-types token-sharing.ts`.




Stream with an OAuth access token authorized for ChatGPT plan usage

```typescript
import OpenAI from "openai";

const accessToken = process.env.ACCESS_TOKEN;
if (!accessToken) throw new Error("Set ACCESS_TOKEN to the user's OAuth token");
const client = new OpenAI({
  apiKey: accessToken,
  baseURL: "https://api.openai.com/v1",
  maxRetries: 0,
});
const stream = await client.responses.create({
  model: "gpt-6.2-sol",
  input: [{ role: "user", content: "Say exactly: Hello, world!" }],
  store: false,
  stream: true,
});
let completed = false;
for await (const event of stream) {
  if (event.type === "response.output_text.delta")
    process.stdout.write(event.delta);
  else if (event.type === "response.failed") {
    throw new Error(
      `Response failed: ${event.response.error?.code ?? "unknown_error"}`
    );
  } else if (event.type === "response.completed") completed = true;
}
if (!completed) throw new Error("Stream ended without response.completed");
process.stdout.write("\n");
```

```python
import os

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["ACCESS_TOKEN"],  # OAuth token sent as Bearer.
    base_url="https://api.openai.com/v1",
    max_retries=0,
)

completed = False
with client.responses.create(
    model="gpt-6.2-sol",
    input=[{"role": "user", "content": "Say exactly: Hello, world!"}],
    store=False,
    stream=True,
) as stream:
    for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="", flush=True)
        elif event.type == "response.failed":
            error = event.response.error
            code = error.code if error else "unknown_error"
            raise RuntimeError(f"Response failed: {code}")
        elif event.type == "response.completed":
            completed = True

if not completed:
    raise RuntimeError("Stream ended without response.completed.")
print()
```


## 3. Wait for completed inference

Treat inference as successful only after receiving `response.completed`. Read through the terminal event. A usage-limit error can arrive as `response.failed` with `subscription_sharing_usage_limit_exceeded` or `subscription_sharing_usage_unavailable` after streaming has begun. Apply the corresponding recovery in [Structured Responses errors](https://developers.openai.com/siwc/token-sharing-open-source/errors-and-recovery#structured-responses-errors). Handle `response.incomplete`, explicit errors, and interrupted streams separately from `response.completed`.