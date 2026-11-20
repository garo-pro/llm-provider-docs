# Time awareness and async input for personal agents

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

If you’re building a personal agent [similar to a dot](https://learn.chatgpt.com/docs/dots), you'll often want to deliver results to the user on a deadline.

Let’s say I have a meeting with a prospective customer in 15 minutes. I ask my agent to research the company and prepare a briefing, leaving me three minutes to read it before the meeting. That’s a pretty typical timeline for my meeting prep during the workday 😬.

The agent notices that I haven’t said whether the meeting is a technical evaluation or a discussion about pricing. That distinction would change which details to investigate and what content to include in the briefing.

The agent decides to ask about the meeting’s focus. While waiting for my reply, how should the agent use the remaining time, incorporate a reply if one arrives, and deliver a useful briefing even if I never answer?

## Time awareness and async input

**Time awareness**, a recent addition to the [Agents API](/api/docs/guides/agents-api/overview), gives the model periodic reminders of the current time so it can adjust its work to the deadline. **[Async function tools](/api/docs/guides/async-tool-calling#how-async-tools-work)** let the agent continue working while the application waits for an answer from the end user.

With 15 minutes until the meeting, the agent has 12 minutes to prepare my briefing. As that deadline approaches, the agent can prioritize checking its strongest findings and writing the briefing over opening another line of research. Alternatively, providing longer timelines lets the model know it can afford to go deeper and perform more research.

A request like “I have a meeting in 15 minutes. Give me a briefing with three minutes left to review it” sets the deadline. Time awareness keeps the model informed as searches and user input are pending.



> Illustration: The agent researches Pied Piper while asking whether the meeting is technical or commercial. A reply focuses the briefing. At 4:10 an unanswered question expires and the agent writes for both meeting goals. It delivers at 4:12, leaving three minutes to read before the 4:15 meeting.



To show how this works, we'll set up an agent to help users prepare for meetings. In the illustration, I ask at 4 p.m. UTC for a briefing by 4:12 p.m., ahead of a 4:15 p.m. meeting. We'll see how to configure the agent, provide timing updates, react to its tool calls, and handle its final result.

You can [skip right to the full example code for your preferred programming language](#complete-example-code).

## Create the agent

To help the user prep for a meeting, our agent needs to research the company, ask for missing meeting context, and track the time left to prepare a briefing. To perform those tasks, the agent will need web search, an asynchronous tool to ask for user input, and time awareness.

We create an agent [with this configuration](/api/docs/guides/agents-api/configuration) using the SDK’s [agent creation method](/api/reference/resources/beta/subresources/agents/methods/create):

- Set `experimental.time_awareness` to `true` to give the model periodic reminders of the current time.
- Enable the built-in [`web_search` tool](/api/docs/guides/agents-api/tools/web-search) in `live` mode so the agent can search the web for current information.
- Define a [function tool](/api/docs/guides/agents-api/tools/functions) named `request_user_input_async`. Set `async` to `true` so the agent can continue working while your application collects the reply.

Set `OPENAI_API_KEY` to a key from a project with Agents API access. Follow the [SDK libraries guide](/api/docs/libraries) to install an official SDK and create the `client` used below. See the [Agents API quickstart prerequisites](/api/docs/guides/agents-api/quickstart#prerequisites) for required permissions.

Create the meeting preparation agent

```typescript
const requests = new AbortController();

const agent = await client.beta.agents.create(
  {
    name: "Personal assistant",
    model: "gpt-6.2-astra",
    instructions:
      "You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. " +
      "Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. " +
      "State important uncertainties.",
    experimental: { time_awareness: true },
    tools: [
      { type: "web_search", mode: "live" },
      {
        type: "function",
        name: "request_user_input_async",
        description:
          "Ask the user for context. The result contains their reply or reports that the question expired.",
        async: true,
        parameters: {
          type: "object",
          properties: {
            question: { type: "string" },
            choices: {
              type: "array",
              items: { type: "string" },
              description:
                "Optional suggested replies to display with the question.",
            },
          },
          required: ["question"],
          additionalProperties: false,
        },
      },
    ],
  },
  { signal: requests.signal }
);
```

```python
async def create_agent(client: AsyncOpenAI):
    return await client.beta.agents.create(
        name="Personal assistant",
        model="gpt-6.2-astra",
        experimental={"time_awareness": True},
        instructions=(
            "You are a personal assistant. Deliver useful, well-sourced answers "
            "within the user's time constraints. Ask for clarification when it "
            "would materially improve the result, and keep making progress while "
            "a reply is pending. State important uncertainties."
        ),
        tools=[
            {"type": "web_search", "mode": "live"},
            {
                "type": "function",
                "name": "request_user_input_async",
                "description": "Ask for missing context while continuing independent work.",
                "async": True,
                "parameters": {
                    "type": "object",
                    "properties": {
                        "question": {"type": "string"},
                        "choices": {"type": "array", "items": {"type": "string"}},
                    },
                    "required": ["question"],
                    "additionalProperties": False,
                },
            },
        ],
    )
```

```go
func createAgent(ctx context.Context, client *openai.Client) (*openai.Agent, error) {
	return client.Beta.Agents.New(ctx, openai.BetaAgentNewParams{
		Name:  openai.String("Personal assistant"),
		Model: "gpt-6.2-astra",
		Instructions: openai.String("You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. " +
			"Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. " +
			"State important uncertainties."),
		Experimental: openai.BetaAgentNewParamsExperimental{TimeAwareness: openai.Bool(true)},
		Tools: []openai.PersistedAgentToolParamUnion{
			{OfParamWebSearch: &openai.PersistedAgentToolParamWebSearch{Mode: "live"}},
			{OfParamFunction: &openai.PersistedAgentToolParamFunction{
				Name:        "request_user_input_async",
				Description: "Ask for missing context while continuing independent work.",
				Async:       openai.Bool(true),
				Parameters: map[string]any{
					"type": "object",
					"properties": map[string]any{
						"question": map[string]any{"type": "string"},
						"choices":  map[string]any{"type": "array", "items": map[string]any{"type": "string"}},
					},
					"required": []string{"question"}, "additionalProperties": false,
				},
			}},
		},
	})
}
```

```java
var agent =
    client
        .beta()
        .agents()
        .create(
            AgentCreateParams.builder()
                .name("Personal assistant")
                .model("gpt-6.2-astra")
                .instructions(
                    "You are a personal assistant. Deliver useful, well-sourced answers"
                        + " within the user's time constraints. Ask for clarification when it"
                        + " would materially improve the result, and keep making progress"
                        + " while a reply is pending. State important uncertainties.")
                .experimental(
                    AgentCreateParams.Experimental.builder().timeAwareness(true).build())
                .addTool(
                    PersistedAgentToolParam.WebSearch.builder()
                        .mode(PersistedAgentToolParam.WebSearch.Mode.LIVE)
                        .build())
                .addTool(
                    PersistedAgentToolParam.Function.builder()
                        .name("request_user_input_async")
                        .description(
                            "Ask the user for context. The result contains their reply or"
                                + " reports that the question expired.")
                        .async(true)
                        .parameters(
                            PersistedAgentToolParam.Function.Parameters.builder()
                                .putAdditionalProperty("type", JsonValue.from("object"))
                                .putAdditionalProperty(
                                    "properties",
                                    JsonValue.from(
                                        Map.of(
                                            "question", Map.of("type", "string"),
                                            "choices",
                                                Map.of(
                                                    "type",
                                                    "array",
                                                    "items",
                                                    Map.of("type", "string"),
                                                    "description",
                                                    "Optional suggested replies to"
                                                        + " display with the question."))))
                                .putAdditionalProperty(
                                    "required", JsonValue.from(List.of("question")))
                                .putAdditionalProperty(
                                    "additionalProperties", JsonValue.from(false))
                                .build())
                        .build())
                .build());
```

```ruby
def create_agent(client)
  client.beta.agents.create(
    name: "Personal assistant",
    model: "gpt-6.2-astra",
    experimental: { time_awareness: true },
    instructions: "You are a personal assistant. Deliver useful, well-sourced answers " \
      "within the user's time constraints. Ask for clarification when it would " \
      "materially improve the result, and keep making progress while a reply is " \
      "pending. State important uncertainties.",
    tools: [
      {
        type: "web_search",
        mode: "live"
      },
      {
        type: "function",
        name: "request_user_input_async",
        description: "Ask for missing context while continuing independent work.",
        async: true,
        parameters: {
          type: "object",
          properties: {
            question: { type: "string" },
            choices: {
              type: "array",
              items: { type: "string" }
            }
          },
          required: ["question"],
          additionalProperties: false
        }
      }
    ]
  )
end
```

```bash
agent=$(curl --fail-with-body --silent --show-error --max-time 30 \
  https://api.openai.com/v1/agents \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "OpenAI-Beta: agents=v1" \
  -H "Content-Type: application/json" \
  --data-binary @- <<'JSON'
{
  "name": "Personal assistant",
  "model": "gpt-6.2-astra",
  "instructions": "You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. State important uncertainties.",
  "experimental": {"time_awareness": true},
  "tools": [
    {"type": "web_search", "mode": "live"},
    {
      "type": "function",
      "name": "request_user_input_async",
      "description": "Ask the user for context. The result contains their reply or reports that the question expired.",
      "async": true,
      "parameters": {
        "type": "object",
        "properties": {
          "question": {"type": "string"},
          "choices": {"type": "array", "items": {"type": "string"}}
        },
        "required": ["question"],
        "additionalProperties": false
      }
    }
  ]
}
JSON
)
agent_id=$(jq -er '.id' <<<"$agent")
```


## Start a task with a deadline

Now we can ask the agent to prepare for a meeting with a customer. The request gives the agent the company, the meeting time, and the briefing deadline, leaving it to decide what to research and whether to ask for clarification.

Start a [session](/api/docs/guides/agents-api/sessions) with the SDK’s [session creation method](/api/reference/resources/beta/subresources/agents/subresources/sessions/methods/create) and stream updates as the agent works.

Your application will need to convert time-based language like “in 15 minutes” and “three minutes left to review” into absolute timestamps so the model and application share the same time data. This code tells the agent the meeting is 15 minutes from now and asks for delivery 12 minutes from now.

Start the meeting preparation session

```typescript
const startedAt = Date.now();
const meetingAt = new Date(startedAt + 15 * 60_000);
const briefingBy = new Date(startedAt + 12 * 60_000);
const replyCutoff = new Date(startedAt + 10 * 60_000);

const stream = await client.beta.agents.sessions.create(
  {
    agent_id: agent.id,
    environment: { type: "none" },
    stream: true,
    input:
      `I am meeting with Pied Piper (example.com) about our customer-support software at ${meetingAt.toISOString()}. ` +
      "Get me up to speed on their business, recent product announcements, and questions I should ask. " +
      `Send me a concise briefing by ${briefingBy.toISOString()} so I have time to read it before the meeting.`,
  },
  { signal: requests.signal }
);
```

```python
now = datetime.now(UTC)
meeting = (now + timedelta(minutes=15)).isoformat(timespec="seconds")
delivery = (now + timedelta(minutes=12)).isoformat(timespec="seconds")
stream = await client.beta.agents.sessions.create(
    agent_id=agent.id,
    environment={"type": "none"},
    input=(
        f"I am meeting with Pied Piper (example.com) about our customer-support software at "
        f"{meeting}. Get me up to speed on their business, recent product "
        f"announcements, and questions I should ask. Send me a concise briefing by "
        f"{delivery} so I have time to read it before the meeting."
    ),
    stream=True,
    timeout=12 * 60,
)
```

```go
events := client.Beta.Agents.Sessions.NewStreaming(streamCtx, openai.BetaAgentSessionNewParams{
	AgentID:     openai.String(agent.ID),
	Environment: openai.EnvironmentParamUnion{OfParamNone: &openai.EnvironmentParamNone{}},
	Input: openai.BetaAgentSessionNewParamsInputUnion{OfString: openai.String(fmt.Sprintf(
		"I am meeting with Pied Piper (example.com) about our customer-support software at %s. "+
			"Get me up to speed on their business, recent product announcements, and questions I should ask. "+
			"Send me a concise briefing by %s so I have time to read it before the meeting.",
		meetingAt.UTC().Format(time.RFC3339), briefingBy.UTC().Format(time.RFC3339)))},
})
```

```java
try (var events =
    client
        .beta()
        .agents()
        .sessions()
        .createStreaming(
            SessionCreateParams.builder()
                .agentId(agent.id())
                .environmentNone()
                .input(
                    "I am meeting with Pied Piper (example.com) about our customer-support software at "
                        + meetingAt
                        + ". Get me up to speed on their business, recent product"
                        + " announcements, and questions I should ask. Send me a concise"
                        + " briefing by "
                        + briefingBy
                        + " so I have time to read it before the meeting.")
                .build(),
            RequestOptions.builder()
                .timeout(
                    Duration.ofMillis(
                        Math.max(
                            1,
                            Duration.between(Instant.now(), briefingBy).toMillis() + 1000)))
                .build())) {
```

```ruby
now = Time.now.utc
meeting = (now + 15 * 60).iso8601
delivery = (now + 12 * 60).iso8601
stream = sessions.create_streaming(
  agent_id: agent_id,
  environment: { type: "none" },
  input: "I am meeting with Pied Piper (example.com) about our customer-support software " \
    "at #{meeting}. Get me up to speed on their business, recent product " \
    "announcements, and questions I should ask. Send me a concise briefing " \
    "by #{delivery} so I have time to read it before the meeting."
)
```

```bash
started=$(date +%s)

meeting_at=$(jq -nr --argjson start "$started" '$start + 900 | strftime("%Y-%m-%dT%H:%M:%SZ")')
briefing_by=$(jq -nr --argjson start "$started" '$start + 720 | strftime("%Y-%m-%dT%H:%M:%SZ")')
request=$(jq -n --arg agent "$agent_id" --arg meeting "$meeting_at" --arg delivery "$briefing_by" '{
  agent_id: $agent,
  environment: {type: "none"},
  stream: true,
  input: ("I am meeting with Pied Piper (example.com) about our customer-support software at " + $meeting + ". Get me up to speed on their business, recent product announcements, and questions I should ask. Send me a concise briefing by " + $delivery + " so I have time to read it before the meeting.")
}')
curl --fail-with-body --silent --show-error --no-buffer \
  https://api.openai.com/v1/agents/sessions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "OpenAI-Beta: agents=v1" \
  -H "Content-Type: application/json" \
  --data-binary "$request" >"$work/events" &
stream_pid=$!
```


## Display a pending question

When the agent asks about the meeting’s focus, the application needs to show the question while company research continues. Collecting a reply in a separate task lets the application keep receiving the agent’s updates.

It's up to your application code to collect user input for clarification when the model asks for it via a tool call.

Watch the [session event stream](/api/docs/guides/agents-api/sessions/events) for [`agent.session.turn.item.done`](/api/reference/resources/beta/subresources/agents/streaming-events) events containing a `request_user_input_async` function call. Read the question and any suggested choices from `item.arguments`. The session’s `required_actions` list describes blocking actions, so waiting for an entry there can miss an asynchronous question.

Save the session ID and the call’s `turn_id` and `call_id` to associate the reply with the question. Track pending questions so each is displayed and handled once, and cancel their input handlers when the agent’s run ends:

Handle a pending question

```typescript
if (event.type === "agent.session.created") {
  sessionId = event.session.id;
}
if (event.type === "agent.session.turn.item.done") {
  const call = event.item;
  if (
    call.type === "function_call" &&
    call.name === "request_user_input_async"
  ) {
    const key = `${call.turn_id}:${call.call_id}`;
    if (!seenCalls.has(key)) {
      seenCalls.add(key);
      // Start collecting the reply without blocking the event stream.
      const task = collectReply(event.session_id, call).catch(
        (error: unknown) => {
          replyError = error;
          requests.abort();
        }
      );
      replyTasks.add(task);
    }
  }
}
```

```python
item = event.item
if (
    isinstance(item, AgentFunctionCallItem)
    and item.name == "request_user_input_async"
    and item.call_id not in questions
):
    questions[item.call_id] = asyncio.create_task(
        respond(
            client,
            event.session_id,
            item,
            replies,
            prompt_lock,
            reply_cutoff,
        )
    )
```

```go
item := event.AsAgentSessionTurnItemDone().Item
if item.Type == "function_call" && item.Name == "request_user_input_async" {
	call := item.AsFunctionCall()
	key := call.TurnID + "/" + call.CallID
	if !seen[key] {
		seen[key] = true
		pending = append(pending, question{call: call})
	}
}
```

```java
event.created().ifPresent(created -> sessionId = created.session().id());
if (event.turnItemDone().isPresent()) {
  var item = event.turnItemDone().get().item();
  if (item.functionCall().isPresent()) {
    var call = item.functionCall().get();
    String key = call.turnId() + ":" + call.callId();
    if (call.name().equals("request_user_input_async") && seenCalls.add(key)) {
      // Collect the reply on another thread while consuming session events.
      replyWorker.submit(
          () -> collectReply(event.turnItemDone().get().sessionId(), call));
    }
  }
}
```

```ruby
item = event.item
if item.is_a?(OpenAI::Models::Beta::AgentFunctionCallItem) &&
   item.name == "request_user_input_async" && !questions.key?(item.call_id)
  questions[item.call_id] = Thread.new(item, event.session_id) do |call, id|
    answer = console_reply(call.arguments, input_lock, cutoff)
    return_result(sessions, id, call, answer)
  rescue OpenAI::Errors::ConflictError
    # The turn can finish while the reply is in flight.
    nil
  end
end
```

```bash
# Buffer terminal replies independently of the event stream.
next_reply=1
(
  number=1
  while IFS= read -r line || [ -n "$line" ]; do
    printf '%s\n' "$line" >"$work/incoming"
    mv "$work/incoming" "$work/reply-$number"
    number=$((number + 1))
  done
) <&4 &
reader_pid=$!

if [ "$event_type" = agent.session.turn.item.done ] && \
  jq -e '.item.type == "function_call" and .item.name == "request_user_input_async"' \
    <<<"$event" >/dev/null; then
  call=$(jq -c '.item' <<<"$event")
  call_id=$(jq -r '.call_id' <<<"$call")
  turn_id=$(jq -r '.turn_id' <<<"$call")
  key="$turn_id:$call_id"
  if ! jq -e --arg key "$key" 'has($key)' <<<"$seen" >/dev/null; then
    seen=$(jq -c --arg key "$key" '.[$key] = true' <<<"$seen")
    pending=$(jq -c --arg key "$key" --argjson call "$call" '.[$key] = $call' <<<"$pending")
    jq -r '.arguments.question, (.arguments.choices[]? // empty)' <<<"$call" >&2
  fi
fi
```


The question arrives as a function-call event:

```json
{
  "type": "agent.session.turn.item.done",
  "item": {
    "type": "function_call",
    "name": "request_user_input_async",
    "arguments": {
      "question": "Is this meeting mainly a technical evaluation or a commercial discussion?",
      "choices": ["Technical evaluation", "Commercial discussion"]
    },
    "turn_id": "turn_789abcd",
    "call_id": "call_012ef34"
  }
}
```

## Return the tool result

The user's answer to the agent's question helps the agent focus the briefing on the right meeting goal. If the user never answers, the agent still needs time to finish writing, so the application closes unanswered questions at 4:10 p.m., two minutes before delivery.

Return one [tool result](/api/docs/guides/agents-api/tools/functions#return-the-result) per pending call while the agent’s run is active:

- **On reply**, set `status` to `answered` and put the typed text or selected choice in `answer`.
- **At the cutoff**, if the same `call_id` is still pending and the user has not replied, return `{"status":"no_answer"}`.

Send the result through the SDK’s [session events method](/api/reference/resources/beta/subresources/agents/subresources/sessions/subresources/events/methods/create), using the saved identifiers to match it to the question. Keep consuming the stream while submitting the result. If the agent finishes early, close the pending question in the interface.

Return the reply or expiration result

```typescript
async function returnResult(
  sessionId: string,
  call: AgentFunctionCallItem,
  answer: string | undefined
) {
  const result = answer
    ? { status: "answered", answer }
    : { status: "no_answer" };
  await client.beta.agents.sessions.events.create(
    sessionId,
    {
      events: [
        {
          type: "agent.session.input.tool_result",
          turn_id: call.turn_id,
          call_id: call.call_id,
          success: true,
          output: JSON.stringify(result),
        },
      ],
    },
    { signal: requests.signal }
  );
}
```

```python
result = (
    {"status": "answered", "answer": answer} if answer else {"status": "no_answer"}
)
await client.beta.agents.sessions.events.create(
    session_id,
    events=[
        {
            "type": "agent.session.input.tool_result",
            "turn_id": call.turn_id,
            "call_id": call.call_id,
            "success": True,
            "output": json.dumps(result),
        }
    ],
)
```

```go
func returnResult(ctx context.Context, client *openai.Client, sessionID string, call openai.AgentFunctionCallItem, answer string) error {
	result := map[string]string{"status": "no_answer"}
	if answer != "" {
		result = map[string]string{"status": "answered", "answer": answer}
	}
	output, err := json.Marshal(result)
	if err != nil {
		return err
	}
	return client.Beta.Agents.Sessions.Events.New(ctx, sessionID, openai.BetaAgentSessionEventNewParams{
		Events: []openai.AgentSessionInputParamUnion{{
			OfParamAgentSessionInputToolResult: &openai.AgentSessionInputParamAgentSessionInputToolResult{
				TurnID: call.TurnID, CallID: call.CallID, Success: true,
				Output: openai.AgentFunctionCallOutputParamUnion{OfString: openai.String(string(output))},
			},
		}},
	})
}
```

```java
private void returnResult(String sessionId, AgentFunctionCallItem call, String answer)
    throws Exception {
  var result =
      answer == null || answer.isBlank()
          ? Map.of("status", "no_answer")
          : Map.of("status", "answered", "answer", answer);
  client
      .beta()
      .agents()
      .sessions()
      .events()
      .create(
          EventCreateParams.builder()
              .sessionId(sessionId)
              .addEvent(
                  AgentSessionInputParam.AgentSessionInputToolResult.builder()
                      .turnId(call.turnId())
                      .callId(call.callId())
                      .success(true)
                      .output(new JsonMapper().writeValueAsString(result))
                      .build())
              .build());
}
```

```ruby
def return_result(sessions, session_id, call, answer)
  result = if answer.nil? || answer.empty?
             { status: "no_answer" }
           else
             {
               status: "answered",
               answer: answer
             }
           end
  sessions.events.create(
    session_id,
    events: [
      {
        type: "agent.session.input.tool_result",
        turn_id: call.turn_id,
        call_id: call.call_id,
        success: true,
        output: JSON.generate(result)
      }
    ]
  )
end
```

```bash
return_result() {
  local turn_id=$1 call_id=$2 answer=$3 result payload
  result=$(jq -nc --arg answer "$answer" '
    if $answer == "" then {status: "no_answer"}
    else {status: "answered", answer: $answer} end')
  payload=$(jq -nc --arg turn "$turn_id" --arg call "$call_id" --arg output "$result" '{
    events: [{type: "agent.session.input.tool_result", turn_id: $turn,
      call_id: $call, success: true, output: $output}]
  }')
  curl --fail-with-body --silent --show-error --max-time 30 \
    "https://api.openai.com/v1/agents/sessions/$session_id/events" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "OpenAI-Beta: agents=v1" \
    -H "Content-Type: application/json" \
    --data-binary "$payload" >/dev/null
}
```


## Retrieve and deliver the answer

Deliver a concise briefing as soon as it is ready, leaving the user at least three minutes to read it before the meeting. If the deadline arrives first, return the latest completed briefing or report that no verified findings are ready.

When `agent.session.turn.completed` arrives, retrieve the saved [items](/api/docs/guides/agents-api/sessions/events#fetch-items-and-turns) with the SDK’s [turn items method](/api/reference/resources/beta/subresources/agents/subresources/sessions/subresources/turns/subresources/items/methods/list). Find the completed agent message whose `phase` is `final_answer`.

If the stream closes early, [list the session’s turns](/api/reference/resources/beta/subresources/agents/subresources/sessions/subresources/turns/methods/list) to check the saved status. See [stream recovery](/api/docs/guides/agents-api/sessions/events#how-to-recover-a-disconnected-stream) for reconnecting and retrieving saved work.

Retrieve the final briefing

```typescript
async function retrieveAnswer(sessionId: string, turnId: string) {
  for await (const item of client.beta.agents.sessions.turns.items.list(
    turnId,
    { session_id: sessionId, order: "desc" },
    { signal: requests.signal }
  )) {
    if (
      item.type === "message" &&
      item.role === "assistant" &&
      item.status === "completed" &&
      item.phase === "final_answer"
    ) {
      return item.content
        .filter((part) => part.type === "output_text")
        .map((part) => part.text)
        .join("\n");
    }
  }
  throw new Error("The completed turn has no final answer");
}
```

```python
def final_text(item: AgentSessionItem | AgentOutputItem) -> str:
    if (
        item.type == "message"
        and item.role == "assistant"
        and item.status == "completed"
        and item.phase == "final_answer"
    ):
        return "\n".join(
            part.text for part in item.content if part.type == "output_text"
        )
    return ""


async def retrieve_answer(client: AsyncOpenAI, session_id: str, turn_id: str) -> str:
    async for item in client.beta.agents.sessions.turns.items.list(
        turn_id, session_id=session_id, order="desc"
    ):
        if text := final_text(item):
            return text
    return ""
```

```go
func retrieveAnswer(ctx context.Context, client *openai.Client, sessionID, turnID string) (string, error) {
	items := client.Beta.Agents.Sessions.Turns.Items.ListAutoPaging(ctx, sessionID, turnID,
		openai.BetaAgentSessionTurnItemListParams{Order: "desc"})
	for items.Next() {
		item := items.Current()
		if item.Type == "message" && item.Role == "assistant" && item.Status == "completed" && item.Phase == "final_answer" {
			var parts []string
			for _, part := range item.AsMessage().Content {
				if part.Type == "output_text" {
					parts = append(parts, part.Text)
				}
			}
			return strings.Join(parts, "\n"), nil
		}
	}
	return "", items.Err()
}
```

```java
private String retrieveAnswer(String sessionId, String turnId) {
  var items =
      client
          .beta()
          .agents()
          .sessions()
          .turns()
          .items()
          .list(
              ItemListParams.builder()
                  .sessionId(sessionId)
                  .turnId(turnId)
                  .order(ItemListParams.Order.DESC)
                  .build());
  for (var item : items.autoPager()) {
    if (item.message().isEmpty()) continue;
    var message = item.message().get();
    if (message.role().toString().equals("assistant")
        && message.status().toString().equals("completed")
        && message.phase().map(phase -> phase.toString().equals("final_answer")).orElse(false)) {
      return message.content().stream()
          .flatMap(part -> part.outputText().stream())
          .map(part -> part.text())
          .collect(Collectors.joining("\n"));
    }
  }
  throw new IllegalStateException("The completed turn has no final answer");
}
```

```ruby
def final_text(item)
  return unless item.type.to_s == "message" && item.role.to_s == "assistant" &&
                item.status.to_s == "completed" && item.phase.to_s == "final_answer"

  text = item.content.select { |part| part.type.to_s == "output_text" }.map(&:text).join("\n")
  text unless text.empty?
end

def retrieve_answer(sessions, session_id, turn_id)
  page = sessions.turns.items.list(turn_id, session_id: session_id, order: "desc")
  page.auto_paging_each do |item|
    text = final_text(item)
    return text if text && !text.empty?
  end
  nil
end
```

```bash
retrieve_answer() {
  local turn_id=$1 page after="" text
  local -a page_args=()
  while :; do
    page=$(curl --fail-with-body --silent --show-error --max-time 30 --get \
      "https://api.openai.com/v1/agents/sessions/$session_id/turns/$turn_id/items" \
      -H "Authorization: Bearer $OPENAI_API_KEY" \
      -H "OpenAI-Beta: agents=v1" \
      --data-urlencode order=desc ${page_args[@]+"${page_args[@]}"})
    text=$(jq -r '[.data[] | select(.type == "message" and .role == "assistant" and
      .status == "completed" and .phase == "final_answer") |
      [.content[] | select(.type == "output_text") | .text] | join("\n")][0] // empty' <<<"$page")
    if [ -n "$text" ]; then printf '%s\n' "$text"; return; fi
    if [ "$(jq -r '.has_more' <<<"$page")" != true ]; then return 1; fi
    after=$(jq -er '.last_id' <<<"$page")
    page_args=(--data-urlencode "after=$after")
  done
}
```


## Complete example code

Each language example below contains the full meeting-preparation workflow in one file, including client setup, asynchronous replies, the delivery timer, and cleanup. Each script reads `OPENAI_API_KEY` from the environment and accepts replies in the terminal.

Complete meeting-preparation example

```typescript
import OpenAI from "openai";
import type { AgentFunctionCallItem } from "openai/resources/beta/agents/agents";
import { createInterface } from "node:readline";

const client = new OpenAI({ timeout: 30_000, maxRetries: 0 });
const startedAt = Date.now();
const meetingAt = new Date(startedAt + 15 * 60_000);
const briefingBy = new Date(startedAt + 12 * 60_000);
const replyCutoff = new Date(startedAt + 10 * 60_000);

const requests = new AbortController();

const terminal = createInterface({ input: process.stdin });
const bufferedReplies: string[] = [];
const waitingReplies = new Map<string, (answer?: string) => void>();
const seenCalls = new Set<string>();
const replyTasks = new Set<Promise<void>>();
let agentId: string | undefined;
let sessionId: string | undefined;
let turnActive = false;
let delivered = false;
let deadlineReached = false;
let finalAnswer = "";
let replyError: unknown;

terminal.on("line", (line) => {
  const waiting = waitingReplies.values().next().value;
  if (waiting) {
    waiting(
      Date.now() < replyCutoff.getTime() ? line.trim() || undefined : undefined
    );
  } else bufferedReplies.push(line.trim());
});

function closeQuestions() {
  for (const finish of waitingReplies.values()) finish();
  terminal.close();
  process.stdin.pause();
}

function deliver(text: string) {
  if (delivered) return;
  delivered = true;
  console.log(text);
}

function readReply(call: AgentFunctionCallItem): Promise<string | undefined> {
  return new Promise((resolve) => {
    const key = `${call.turn_id}:${call.call_id}`;
    let timer: ReturnType<typeof setTimeout> | undefined;
    const finish = (answer?: string) => {
      clearTimeout(timer);
      waitingReplies.delete(key);
      resolve(answer);
    };
    waitingReplies.set(key, finish);
    timer = setTimeout(finish, Math.max(0, replyCutoff.getTime() - Date.now()));
    if (Date.now() >= replyCutoff.getTime()) finish();
    else if (bufferedReplies.length)
      finish(bufferedReplies.shift() || undefined);
  });
}

async function returnResult(
  sessionId: string,
  call: AgentFunctionCallItem,
  answer: string | undefined
) {
  const result = answer
    ? { status: "answered", answer }
    : { status: "no_answer" };
  await client.beta.agents.sessions.events.create(
    sessionId,
    {
      events: [
        {
          type: "agent.session.input.tool_result",
          turn_id: call.turn_id,
          call_id: call.call_id,
          success: true,
          output: JSON.stringify(result),
        },
      ],
    },
    { signal: requests.signal }
  );
}

async function collectReply(sessionId: string, call: AgentFunctionCallItem) {
  const arguments_ = call.arguments;
  if (
    typeof arguments_ !== "object" ||
    arguments_ === null ||
    !("question" in arguments_) ||
    typeof arguments_.question !== "string"
  ) {
    throw new Error("The question tool did not provide a question");
  }
  console.error(arguments_.question);
  if ("choices" in arguments_ && Array.isArray(arguments_.choices)) {
    console.error(arguments_.choices.join(" / "));
  }
  const answer = await readReply(call);
  if (turnActive && !requests.signal.aborted) {
    await returnResult(sessionId, call, answer);
  }
}

async function retrieveAnswer(sessionId: string, turnId: string) {
  for await (const item of client.beta.agents.sessions.turns.items.list(
    turnId,
    { session_id: sessionId, order: "desc" },
    { signal: requests.signal }
  )) {
    if (
      item.type === "message" &&
      item.role === "assistant" &&
      item.status === "completed" &&
      item.phase === "final_answer"
    ) {
      return item.content
        .filter((part) => part.type === "output_text")
        .map((part) => part.text)
        .join("\n");
    }
  }
  throw new Error("The completed turn has no final answer");
}

const deliveryTimer = setTimeout(
  () => {
    deadlineReached = true;
    deliver(
      finalAnswer || "No results were verified by the briefing deadline."
    );
    turnActive = false;
    closeQuestions();
    requests.abort();
  },
  Math.max(0, briefingBy.getTime() - Date.now())
);

try {
  const agent = await client.beta.agents.create(
    {
      name: "Personal assistant",
      model: "gpt-6.2-astra",
      instructions:
        "You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. " +
        "Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. " +
        "State important uncertainties.",
      experimental: { time_awareness: true },
      tools: [
        { type: "web_search", mode: "live" },
        {
          type: "function",
          name: "request_user_input_async",
          description:
            "Ask the user for context. The result contains their reply or reports that the question expired.",
          async: true,
          parameters: {
            type: "object",
            properties: {
              question: { type: "string" },
              choices: {
                type: "array",
                items: { type: "string" },
                description:
                  "Optional suggested replies to display with the question.",
              },
            },
            required: ["question"],
            additionalProperties: false,
          },
        },
      ],
    },
    { signal: requests.signal }
  );
  agentId = agent.id;
  const stream = await client.beta.agents.sessions.create(
    {
      agent_id: agent.id,
      environment: { type: "none" },
      stream: true,
      input:
        `I am meeting with Pied Piper (example.com) about our customer-support software at ${meetingAt.toISOString()}. ` +
        "Get me up to speed on their business, recent product announcements, and questions I should ask. " +
        `Send me a concise briefing by ${briefingBy.toISOString()} so I have time to read it before the meeting.`,
    },
    { signal: requests.signal }
  );
  turnActive = true;
  for await (const event of stream) {
    if (event.type === "agent.session.created") {
      sessionId = event.session.id;
    }
    if (event.type === "agent.session.turn.item.done") {
      const call = event.item;
      if (
        call.type === "function_call" &&
        call.name === "request_user_input_async"
      ) {
        const key = `${call.turn_id}:${call.call_id}`;
        if (!seenCalls.has(key)) {
          seenCalls.add(key);
          // Start collecting the reply without blocking the event stream.
          const task = collectReply(event.session_id, call).catch(
            (error: unknown) => {
              replyError = error;
              requests.abort();
            }
          );
          replyTasks.add(task);
        }
      }
    }
    if (
      event.type === "agent.session.turn.item.done" &&
      event.item.type === "message" &&
      event.item.role === "assistant" &&
      event.item.phase === "final_answer" &&
      event.item.status === "completed"
    ) {
      finalAnswer = event.item.content.map((part) => part.text).join("\n");
    }
    if (event.type === "agent.session.turn.completed") {
      turnActive = false;
      closeQuestions();
      finalAnswer = await retrieveAnswer(event.session_id, event.turn.id);
      deliver(finalAnswer);
      break;
    }
    if (
      event.type === "agent.session.turn.failed" ||
      event.type === "agent.session.turn.cancelled" ||
      event.type === "agent.session.failed" ||
      event.type === "agent.session.environment.failed" ||
      event.type === "error"
    ) {
      throw new Error(`Agent lifecycle failure: ${event.type}`);
    }
  }
  if (replyError) throw replyError;
  if (!delivered && !deadlineReached) {
    throw new Error("The stream closed before the turn completed");
  }
} catch (error) {
  if (!deadlineReached) throw replyError || error;
} finally {
  clearTimeout(deliveryTimer);
  const mustCancel = turnActive || deadlineReached;
  turnActive = false;
  closeQuestions();
  requests.abort();
  await Promise.allSettled(replyTasks);
  try {
    if (sessionId) {
      try {
        if (mustCancel) {
          await client.beta.agents.sessions.events.create(sessionId, {
            events: [{ type: "agent.session.input.cancel" }],
          });
        }
      } finally {
        // Cancellation can briefly leave the session busy.
        await client.beta.agents.sessions.delete(sessionId, {
          maxRetries: 5,
          timeout: 10_000,
        });
      }
    }
  } finally {
    if (agentId) await client.beta.agents.delete(agentId);
  }
}

```

```python
import asyncio
import json
import os
import sys
from contextlib import AsyncExitStack, suppress
from datetime import UTC, datetime, timedelta
from threading import Thread

from openai import AsyncOpenAI, ConflictError
from openai.types.beta import AgentFunctionCallItem, AgentOutputItem, AgentSessionItem

client = AsyncOpenAI(max_retries=0)


async def create_agent(client: AsyncOpenAI):
    return await client.beta.agents.create(
        name="Personal assistant",
        model="gpt-6.2-astra",
        experimental={"time_awareness": True},
        instructions=(
            "You are a personal assistant. Deliver useful, well-sourced answers "
            "within the user's time constraints. Ask for clarification when it "
            "would materially improve the result, and keep making progress while "
            "a reply is pending. State important uncertainties."
        ),
        tools=[
            {"type": "web_search", "mode": "live"},
            {
                "type": "function",
                "name": "request_user_input_async",
                "description": "Ask for missing context while continuing independent work.",
                "async": True,
                "parameters": {
                    "type": "object",
                    "properties": {
                        "question": {"type": "string"},
                        "choices": {"type": "array", "items": {"type": "string"}},
                    },
                    "required": ["question"],
                    "additionalProperties": False,
                },
            },
        ],
    )




def console_replies() -> asyncio.Queue[str]:
    loop = asyncio.get_running_loop()
    replies: asyncio.Queue[str] = asyncio.Queue()

    def read_lines():
        with os.fdopen(
            os.dup(sys.stdin.fileno()), encoding=sys.stdin.encoding
        ) as terminal:
            for line in terminal:
                try:
                    loop.call_soon_threadsafe(replies.put_nowait, line.strip())
                except RuntimeError:
                    return  # The session and its event loop have closed.

    # A pending terminal read must not keep the program open after delivery.
    Thread(target=read_lines, daemon=True).start()
    return replies


async def respond(client, session_id, call, replies, prompt_lock, cutoff):
    loop = asyncio.get_running_loop()
    try:
        async with asyncio.timeout_at(cutoff), prompt_lock:
            if loop.time() >= cutoff:
                raise TimeoutError
            print(call.arguments["question"], file=sys.stderr, flush=True)
            for choice in call.arguments.get("choices", []):
                print(f"  {choice}", file=sys.stderr, flush=True)
            reply = await replies.get()
            answer = reply if loop.time() < cutoff else ""
    except TimeoutError:
        answer = ""

    result = (
        {"status": "answered", "answer": answer} if answer else {"status": "no_answer"}
    )
    await client.beta.agents.sessions.events.create(
        session_id,
        events=[
            {
                "type": "agent.session.input.tool_result",
                "turn_id": call.turn_id,
                "call_id": call.call_id,
                "success": True,
                "output": json.dumps(result),
            }
        ],
    )


def final_text(item: AgentSessionItem | AgentOutputItem) -> str:
    if (
        item.type == "message"
        and item.role == "assistant"
        and item.status == "completed"
        and item.phase == "final_answer"
    ):
        return "\n".join(
            part.text for part in item.content if part.type == "output_text"
        )
    return ""


async def retrieve_answer(client: AsyncOpenAI, session_id: str, turn_id: str) -> str:
    async for item in client.beta.agents.sessions.turns.items.list(
        turn_id, session_id=session_id, order="desc"
    ):
        if text := final_text(item):
            return text
    return ""




async def main():
    async with client, AsyncExitStack() as cleanup:
        agent = await create_agent(client)
        cleanup.push_async_callback(client.beta.agents.delete, agent.id)
        sessions = client.beta.agents.sessions
        session_id = None
        active = False
        saved_answer = ""
        questions = {}
        prompt_lock = asyncio.Lock()
        replies = console_replies()
        loop = asyncio.get_running_loop()
        started = loop.time()
        reply_cutoff = started + 10 * 60
        try:
            async with asyncio.timeout_at(started + 12 * 60):
                now = datetime.now(UTC)
                meeting = (now + timedelta(minutes=15)).isoformat(timespec="seconds")
                delivery = (now + timedelta(minutes=12)).isoformat(timespec="seconds")
                stream = await client.beta.agents.sessions.create(
                    agent_id=agent.id,
                    environment={"type": "none"},
                    input=(
                        f"I am meeting with Pied Piper (example.com) about our customer-support software at "
                        f"{meeting}. Get me up to speed on their business, recent product "
                        f"announcements, and questions I should ask. Send me a concise briefing by "
                        f"{delivery} so I have time to read it before the meeting."
                    ),
                    stream=True,
                    timeout=12 * 60,
                )
                async with stream:
                    async for event in stream:
                        if event.type == "agent.session.created":
                            session_id = event.session.id
                            # A cancelled turn can take a moment to release its session.
                            cleanup_sessions = client.with_options(
                                max_retries=4, timeout=10
                            ).beta.agents.sessions
                            cleanup.push_async_callback(
                                cleanup_sessions.delete, session_id
                            )
                            active = True
                        elif event.type == "agent.session.turn.item.done":
                            item = event.item
                            if (
                                isinstance(item, AgentFunctionCallItem)
                                and item.name == "request_user_input_async"
                                and item.call_id not in questions
                            ):
                                questions[item.call_id] = asyncio.create_task(
                                    respond(
                                        client,
                                        event.session_id,
                                        item,
                                        replies,
                                        prompt_lock,
                                        reply_cutoff,
                                    )
                                )
                            saved_answer = final_text(item) or saved_answer
                        elif event.type == "agent.session.turn.completed":
                            active = False
                            for question in questions.values():
                                question.cancel()
                            # Allow the creation stream to finish releasing its connection.
                            with suppress(TimeoutError):
                                async with asyncio.timeout(1):
                                    async for _ in stream:
                                        continue
                            answer = await retrieve_answer(
                                client, event.session_id, event.turn.id
                            )
                            print(
                                answer
                                or saved_answer
                                or "No verified findings were available.",
                                flush=True,
                            )
                            return
                        elif event.type in (
                            "agent.session.failed",
                            "agent.session.environment.failed",
                            "agent.session.turn.failed",
                            "agent.session.turn.cancelled",
                            "error",
                        ):
                            active = False
                            raise RuntimeError(
                                f"Meeting briefing stopped: {event.type}"
                            )
                        for question in questions.values():
                            if question.done():
                                error = question.exception()
                                # A finished turn can reject a reply still in flight.
                                if error is not None and not isinstance(
                                    error, ConflictError
                                ):
                                    raise error
                raise RuntimeError("The event stream closed before the turn completed.")
        except TimeoutError:
            print(
                saved_answer or "No verified findings were available by the deadline.",
                flush=True,
            )
        finally:
            for question in questions.values():
                question.cancel()
            await asyncio.gather(*questions.values(), return_exceptions=True)
            if active and session_id:
                await sessions.events.create(
                    session_id, events=[{"type": "agent.session.input.cancel"}]
                )


if __name__ == "__main__":
    asyncio.run(main())

```

```go
package main

import (
	"bufio"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"strings"
	"time"

	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/option"
)

func createAgent(ctx context.Context, client *openai.Client) (*openai.Agent, error) {
	return client.Beta.Agents.New(ctx, openai.BetaAgentNewParams{
		Name:  openai.String("Personal assistant"),
		Model: "gpt-6.2-astra",
		Instructions: openai.String("You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. " +
			"Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. " +
			"State important uncertainties."),
		Experimental: openai.BetaAgentNewParamsExperimental{TimeAwareness: openai.Bool(true)},
		Tools: []openai.PersistedAgentToolParamUnion{
			{OfParamWebSearch: &openai.PersistedAgentToolParamWebSearch{Mode: "live"}},
			{OfParamFunction: &openai.PersistedAgentToolParamFunction{
				Name:        "request_user_input_async",
				Description: "Ask for missing context while continuing independent work.",
				Async:       openai.Bool(true),
				Parameters: map[string]any{
					"type": "object",
					"properties": map[string]any{
						"question": map[string]any{"type": "string"},
						"choices":  map[string]any{"type": "array", "items": map[string]any{"type": "string"}},
					},
					"required": []string{"question"}, "additionalProperties": false,
				},
			}},
		},
	})
}


func returnResult(ctx context.Context, client *openai.Client, sessionID string, call openai.AgentFunctionCallItem, answer string) error {
	result := map[string]string{"status": "no_answer"}
	if answer != "" {
		result = map[string]string{"status": "answered", "answer": answer}
	}
	output, err := json.Marshal(result)
	if err != nil {
		return err
	}
	return client.Beta.Agents.Sessions.Events.New(ctx, sessionID, openai.BetaAgentSessionEventNewParams{
		Events: []openai.AgentSessionInputParamUnion{{
			OfParamAgentSessionInputToolResult: &openai.AgentSessionInputParamAgentSessionInputToolResult{
				TurnID: call.TurnID, CallID: call.CallID, Success: true,
				Output: openai.AgentFunctionCallOutputParamUnion{OfString: openai.String(string(output))},
			},
		}},
	})
}


func retrieveAnswer(ctx context.Context, client *openai.Client, sessionID, turnID string) (string, error) {
	items := client.Beta.Agents.Sessions.Turns.Items.ListAutoPaging(ctx, sessionID, turnID,
		openai.BetaAgentSessionTurnItemListParams{Order: "desc"})
	for items.Next() {
		item := items.Current()
		if item.Type == "message" && item.Role == "assistant" && item.Status == "completed" && item.Phase == "final_answer" {
			var parts []string
			for _, part := range item.AsMessage().Content {
				if part.Type == "output_text" {
					parts = append(parts, part.Text)
				}
			}
			return strings.Join(parts, "\n"), nil
		}
	}
	return "", items.Err()
}


func consoleReplies(ctx context.Context) <-chan string {
	lines := make(chan string)
	go func() {
		defer close(lines)
		scanner := bufio.NewScanner(os.Stdin)
		for scanner.Scan() {
			select {
			case lines <- strings.TrimSpace(scanner.Text()):
			case <-ctx.Done():
				return
			}
		}
	}()
	return lines
}

type question struct {
	call  openai.AgentFunctionCallItem
	shown bool
}

func (q *question) show() error {
	args, _ := q.call.Arguments.(map[string]any)
	text, _ := args["question"].(string)
	if text == "" {
		return errors.New("the clarification tool did not supply a question")
	}
	fmt.Fprintln(os.Stderr, text)
	choices, _ := args["choices"].([]any)
	for _, choice := range choices {
		fmt.Fprintln(os.Stderr, " ", choice)
	}
	q.shown = true
	return nil
}

func run() (err error) {
	started := time.Now()
	replyCutoff := started.Add(10 * time.Minute)
	briefingBy := started.Add(12 * time.Minute)
	meetingAt := started.Add(15 * time.Minute)
	ctx, cancel := context.WithDeadline(context.Background(), briefingBy)
	defer cancel()
	client := openai.NewClient(option.WithMaxRetries(0))
	var agentID, sessionID, finalText string
	var active, delivered bool
	defer func() {
		if !delivered && errors.Is(ctx.Err(), context.DeadlineExceeded) {
			if finalText == "" {
				finalText = "No verified findings were available by the deadline."
			}
			fmt.Println(finalText)
			err = nil
		}
		cleanup, finish := context.WithTimeout(context.Background(), time.Minute)
		defer finish()
		if sessionID != "" {
			if active {
				err = errors.Join(err, client.Beta.Agents.Sessions.Events.New(cleanup, sessionID, openai.BetaAgentSessionEventNewParams{
					Events: []openai.AgentSessionInputParamUnion{{OfParamAgentSessionInputCancel: &openai.AgentSessionInputParamAgentSessionInputCancel{}}},
				}))
			}
			// A cancelled turn can take a moment to release its session.
			_, deleteErr := client.Beta.Agents.Sessions.Delete(cleanup, sessionID, option.WithMaxRetries(4))
			err = errors.Join(err, deleteErr)
		}
		if agentID != "" {
			_, deleteErr := client.Beta.Agents.Delete(cleanup, agentID)
			err = errors.Join(err, deleteErr)
		}
	}()

	agent, err := createAgent(ctx, &client)
	if err != nil {
		return err
	}
	agentID = agent.ID
	streamCtx, stopStream := context.WithCancel(ctx)
	events := client.Beta.Agents.Sessions.NewStreaming(streamCtx, openai.BetaAgentSessionNewParams{
		AgentID:     openai.String(agent.ID),
		Environment: openai.EnvironmentParamUnion{OfParamNone: &openai.EnvironmentParamNone{}},
		Input: openai.BetaAgentSessionNewParamsInputUnion{OfString: openai.String(fmt.Sprintf(
			"I am meeting with Pied Piper (example.com) about our customer-support software at %s. "+
				"Get me up to speed on their business, recent product announcements, and questions I should ask. "+
				"Send me a concise briefing by %s so I have time to read it before the meeting.",
			meetingAt.UTC().Format(time.RFC3339), briefingBy.UTC().Format(time.RFC3339)))},
	})
	incoming := make(chan openai.AgentSessionEventUnion)
	readerDone := make(chan struct{})
	go func() {
		defer close(readerDone)
		defer close(incoming)
		defer events.Close()
		for events.Next() {
			select {
			case incoming <- events.Current():
			case <-streamCtx.Done():
				return
			}
		}
	}()
	defer func() {
		stopStream()
		<-readerDone
	}()

	lines := consoleReplies(ctx)
	cutoffTimer := time.NewTimer(time.Until(replyCutoff))
	defer cutoffTimer.Stop()
	seen := map[string]bool{}
	var pending []question
	for {
		var reply <-chan string
		if len(pending) != 0 {
			if !time.Now().Before(replyCutoff) || lines == nil {
				if err := returnResult(ctx, &client, sessionID, pending[0].call, ""); err != nil {
					return err
				}
				pending = pending[1:]
				continue
			}
			if !pending[0].shown {
				if err := pending[0].show(); err != nil {
					return err
				}
			}
			reply = lines
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-cutoffTimer.C:
			continue
		case answer, open := <-reply:
			if !open {
				lines = nil
			}
			if !time.Now().Before(replyCutoff) {
				answer = ""
			}
			if err := returnResult(ctx, &client, sessionID, pending[0].call, answer); err != nil {
				return err
			}
			pending = pending[1:]
		case event, open := <-incoming:
			if !open {
				if err := events.Err(); err != nil {
					return err
				}
				return errors.New("the session stream closed before the turn finished")
			}
			switch event.Type {
			case "agent.session.created":
				sessionID, active = event.Session.ID, true
			case "agent.session.turn.item.done":
				item := event.AsAgentSessionTurnItemDone().Item
				if item.Type == "function_call" && item.Name == "request_user_input_async" {
					call := item.AsFunctionCall()
					key := call.TurnID + "/" + call.CallID
					if !seen[key] {
						seen[key] = true
						pending = append(pending, question{call: call})
					}
				}
				if item.Type == "message" && item.Status == "completed" && item.Phase == "final_answer" {
					var parts []string
					for _, part := range item.AsMessage().Content {
						parts = append(parts, part.Text)
					}
					finalText = strings.Join(parts, "\n")
				}
			case "agent.session.turn.completed":
				active = false
				answer, err := retrieveAnswer(ctx, &client, sessionID, event.Turn.ID)
				if err != nil {
					return err
				}
				if answer == "" {
					answer = finalText
				}
				if answer == "" {
					return errors.New("the completed turn has no final briefing")
				}
				fmt.Println(answer)
				delivered = true
				return nil
			case "agent.session.turn.failed", "agent.session.turn.cancelled", "agent.session.failed", "agent.session.environment.failed", "error":
				return fmt.Errorf("meeting briefing stopped: %s", event.Type)
			}
		}
	}
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

```

```java
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.core.JsonValue;
import com.openai.core.RequestOptions;
import com.openai.core.http.StreamResponse;
import com.openai.models.beta.agents.AgentCreateParams;
import com.openai.models.beta.agents.AgentFunctionCallItem;
import com.openai.models.beta.agents.AgentSessionEvent;
import com.openai.models.beta.agents.AgentSessionInputParam;
import com.openai.models.beta.agents.PersistedAgentToolParam;
import com.openai.models.beta.agents.sessions.SessionCreateParams;
import com.openai.models.beta.agents.sessions.events.EventCreateParams;
import com.openai.models.beta.agents.sessions.turns.items.ItemListParams;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.time.Duration;
import java.time.Instant;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicReference;
import java.util.stream.Collectors;

public final class MeetingPreparation {
  public record Question(String question, List<String> choices) {}

  private final OpenAIClient client =
      OpenAIOkHttpClient.builder().fromEnv().timeout(Duration.ofSeconds(30)).maxRetries(0).build();
  private final Instant startedAt = Instant.now();
  private final Instant meetingAt = startedAt.plus(Duration.ofMinutes(15));
  private final Instant briefingBy = startedAt.plus(Duration.ofMinutes(12));
  private final Instant replyCutoff = startedAt.plus(Duration.ofMinutes(10));
  private final BlockingQueue<String> replies = new LinkedBlockingQueue<>();
  private final Set<String> seenCalls = new HashSet<>();
  private final AtomicBoolean active = new AtomicBoolean();
  private final AtomicBoolean expired = new AtomicBoolean();
  private final AtomicBoolean delivered = new AtomicBoolean();
  private final AtomicReference<StreamResponse<AgentSessionEvent>> streamRef =
      new AtomicReference<>();
  private final AtomicReference<RuntimeException> replyError = new AtomicReference<>();
  private final java.util.concurrent.ExecutorService replyWorker =
      Executors.newSingleThreadExecutor();
  private final java.util.concurrent.ScheduledExecutorService timers =
      Executors.newSingleThreadScheduledExecutor();
  private String agentId;
  private String sessionId;
  private volatile String finalAnswer = "";

  public static void main(String[] args) throws Exception {
    new MeetingPreparation().run();
  }

  private void run() throws Exception {
    startInputReader();
    var deliveryTimer =
        timers.schedule(
            () -> {
              expired.set(true);
              deliver(
                  finalAnswer.isBlank()
                      ? "No results were verified by the briefing deadline."
                      : finalAnswer);
              active.set(false);
              closeStream();
            },
            Math.max(0, Duration.between(Instant.now(), briefingBy).toMillis()),
            TimeUnit.MILLISECONDS);
    try {
      var agent =
          client
              .beta()
              .agents()
              .create(
                  AgentCreateParams.builder()
                      .name("Personal assistant")
                      .model("gpt-6.2-astra")
                      .instructions(
                          "You are a personal assistant. Deliver useful, well-sourced answers"
                              + " within the user's time constraints. Ask for clarification when it"
                              + " would materially improve the result, and keep making progress"
                              + " while a reply is pending. State important uncertainties.")
                      .experimental(
                          AgentCreateParams.Experimental.builder().timeAwareness(true).build())
                      .addTool(
                          PersistedAgentToolParam.WebSearch.builder()
                              .mode(PersistedAgentToolParam.WebSearch.Mode.LIVE)
                              .build())
                      .addTool(
                          PersistedAgentToolParam.Function.builder()
                              .name("request_user_input_async")
                              .description(
                                  "Ask the user for context. The result contains their reply or"
                                      + " reports that the question expired.")
                              .async(true)
                              .parameters(
                                  PersistedAgentToolParam.Function.Parameters.builder()
                                      .putAdditionalProperty("type", JsonValue.from("object"))
                                      .putAdditionalProperty(
                                          "properties",
                                          JsonValue.from(
                                              Map.of(
                                                  "question", Map.of("type", "string"),
                                                  "choices",
                                                      Map.of(
                                                          "type",
                                                          "array",
                                                          "items",
                                                          Map.of("type", "string"),
                                                          "description",
                                                          "Optional suggested replies to"
                                                              + " display with the question."))))
                                      .putAdditionalProperty(
                                          "required", JsonValue.from(List.of("question")))
                                      .putAdditionalProperty(
                                          "additionalProperties", JsonValue.from(false))
                                      .build())
                              .build())
                      .build());
      agentId = agent.id();
      if (expired.get()) return;
      try (var events =
          client
              .beta()
              .agents()
              .sessions()
              .createStreaming(
                  SessionCreateParams.builder()
                      .agentId(agent.id())
                      .environmentNone()
                      .input(
                          "I am meeting with Pied Piper (example.com) about our customer-support software at "
                              + meetingAt
                              + ". Get me up to speed on their business, recent product"
                              + " announcements, and questions I should ask. Send me a concise"
                              + " briefing by "
                              + briefingBy
                              + " so I have time to read it before the meeting.")
                      .build(),
                  RequestOptions.builder()
                      .timeout(
                          Duration.ofMillis(
                              Math.max(
                                  1,
                                  Duration.between(Instant.now(), briefingBy).toMillis() + 1000)))
                      .build())) {
        streamRef.set(events);
        if (expired.get()) return;
        active.set(true);
        var iterator = events.stream().iterator();
        while (!expired.get() && iterator.hasNext()) {
          var event = iterator.next();
          event.created().ifPresent(created -> sessionId = created.session().id());
          if (event.turnItemDone().isPresent()) {
            var item = event.turnItemDone().get().item();
            if (item.functionCall().isPresent()) {
              var call = item.functionCall().get();
              String key = call.turnId() + ":" + call.callId();
              if (call.name().equals("request_user_input_async") && seenCalls.add(key)) {
                // Collect the reply on another thread while consuming session events.
                replyWorker.submit(
                    () -> collectReply(event.turnItemDone().get().sessionId(), call));
              }
            }
          }
          event
              .turnItemDone()
              .flatMap(done -> done.item().message())
              .filter(message -> message.status().toString().equals("completed"))
              .filter(
                  message ->
                      message
                          .phase()
                          .map(phase -> phase.toString().equals("final_answer"))
                          .orElse(false))
              .ifPresent(
                  message ->
                      finalAnswer =
                          message.content().stream()
                              .map(part -> part.text())
                              .collect(Collectors.joining("\n")));
          if (event.turnCompleted().isPresent()) {
            active.set(false);
            replyWorker.shutdownNow();
            var completed = event.turnCompleted().get();
            finalAnswer = retrieveAnswer(completed.sessionId(), completed.turn().id());
            deliver(finalAnswer);
            break;
          }
          if (event.turnFailed().isPresent()
              || event.turnCancelled().isPresent()
              || event.failed().isPresent()
              || event.environmentFailed().isPresent()
              || event.error().isPresent()) {
            throw new IllegalStateException("Agent lifecycle failure: " + event);
          }
        }
      }
      if (replyError.get() != null) throw replyError.get();
      if (!delivered.get())
        throw new IllegalStateException("Stream closed before the turn completed");
    } catch (RuntimeException error) {
      if (!expired.get()) throw error;
    } finally {
      deliveryTimer.cancel(false);
      boolean mustCancel = active.getAndSet(false) || expired.get();
      replyWorker.shutdownNow();
      timers.shutdownNow();
      closeStream();
      try {
        if (sessionId != null) {
          try {
            if (mustCancel) {
              client
                  .beta()
                  .agents()
                  .sessions()
                  .events()
                  .create(
                      EventCreateParams.builder()
                          .sessionId(sessionId)
                          .addEventAgentSessionInputCancel()
                          .build());
            }
          } finally {
            // Cancellation settles asynchronously; retry a temporary deletion conflict.
            client
                .withOptions(options -> options.maxRetries(5).timeout(Duration.ofSeconds(10)))
                .beta()
                .agents()
                .sessions()
                .delete(sessionId);
          }
        }
      } finally {
        if (agentId != null) client.beta().agents().delete(agentId);
      }
    }
  }

  private void collectReply(String sessionId, AgentFunctionCallItem call) {
    try {
      var question = call._arguments().convert(Question.class);
      System.err.println(question.question());
      if (question.choices() != null) System.err.println(String.join(" / ", question.choices()));
      long remaining = Math.max(0, Duration.between(Instant.now(), replyCutoff).toMillis());
      String answer = remaining == 0 ? null : replies.poll(remaining, TimeUnit.MILLISECONDS);
      if (Instant.now().isAfter(replyCutoff)) answer = null;
      if (active.get() && !expired.get()) returnResult(sessionId, call, answer);
    } catch (InterruptedException ignored) {
      Thread.currentThread().interrupt();
    } catch (Exception error) {
      if (active.get() && !expired.get()) {
        replyError.set(new IllegalStateException("Could not return the reply", error));
        closeStream();
      }
    }
  }

  private void returnResult(String sessionId, AgentFunctionCallItem call, String answer)
      throws Exception {
    var result =
        answer == null || answer.isBlank()
            ? Map.of("status", "no_answer")
            : Map.of("status", "answered", "answer", answer);
    client
        .beta()
        .agents()
        .sessions()
        .events()
        .create(
            EventCreateParams.builder()
                .sessionId(sessionId)
                .addEvent(
                    AgentSessionInputParam.AgentSessionInputToolResult.builder()
                        .turnId(call.turnId())
                        .callId(call.callId())
                        .success(true)
                        .output(new JsonMapper().writeValueAsString(result))
                        .build())
                .build());
  }


  private String retrieveAnswer(String sessionId, String turnId) {
    var items =
        client
            .beta()
            .agents()
            .sessions()
            .turns()
            .items()
            .list(
                ItemListParams.builder()
                    .sessionId(sessionId)
                    .turnId(turnId)
                    .order(ItemListParams.Order.DESC)
                    .build());
    for (var item : items.autoPager()) {
      if (item.message().isEmpty()) continue;
      var message = item.message().get();
      if (message.role().toString().equals("assistant")
          && message.status().toString().equals("completed")
          && message.phase().map(phase -> phase.toString().equals("final_answer")).orElse(false)) {
        return message.content().stream()
            .flatMap(part -> part.outputText().stream())
            .map(part -> part.text())
            .collect(Collectors.joining("\n"));
      }
    }
    throw new IllegalStateException("The completed turn has no final answer");
  }


  private void deliver(String text) {
    if (delivered.compareAndSet(false, true)) System.out.println(text);
  }

  private void closeStream() {
    var stream = streamRef.getAndSet(null);
    if (stream != null) stream.close();
  }

  private void startInputReader() {
    var input =
        new Thread(
            () -> {
              try {
                var reader = new BufferedReader(new InputStreamReader(System.in));
                String line;
                while ((line = reader.readLine()) != null) replies.add(line.trim());
              } catch (java.io.IOException error) {
                System.err.println("Input closed: " + error.getMessage());
              }
            });
    input.setDaemon(true);
    input.start();
  }
}

```

```ruby
require "openai"
require "json"
require "time"
require "timeout"

def create_agent(client)
  client.beta.agents.create(
    name: "Personal assistant",
    model: "gpt-6.2-astra",
    experimental: { time_awareness: true },
    instructions: "You are a personal assistant. Deliver useful, well-sourced answers " \
      "within the user's time constraints. Ask for clarification when it would " \
      "materially improve the result, and keep making progress while a reply is " \
      "pending. State important uncertainties.",
    tools: [
      {
        type: "web_search",
        mode: "live"
      },
      {
        type: "function",
        name: "request_user_input_async",
        description: "Ask for missing context while continuing independent work.",
        async: true,
        parameters: {
          type: "object",
          properties: {
            question: { type: "string" },
            choices: {
              type: "array",
              items: { type: "string" }
            }
          },
          required: ["question"],
          additionalProperties: false
        }
      }
    ]
  )
end

def monotonic_time
  Process.clock_gettime(Process::CLOCK_MONOTONIC)
end

def console_reply(arguments, input_lock, cutoff)
  remaining = cutoff - monotonic_time
  return if remaining <= 0

  Timeout.timeout(remaining) do
    input_lock.synchronize do
      return if monotonic_time >= cutoff

      question = arguments.transform_keys(&:to_s)
      warn question.fetch("question")
      Array(question["choices"]).each { |choice| warn "  #{choice}" }
      line = $stdin.gets
      sleep([cutoff - monotonic_time, 0].max) if line.nil?
      line&.strip if monotonic_time < cutoff
    end
  end
rescue Timeout::Error
  nil
end

def return_result(sessions, session_id, call, answer)
  result = if answer.nil? || answer.empty?
             { status: "no_answer" }
           else
             {
               status: "answered",
               answer: answer
             }
           end
  sessions.events.create(
    session_id,
    events: [
      {
        type: "agent.session.input.tool_result",
        turn_id: call.turn_id,
        call_id: call.call_id,
        success: true,
        output: JSON.generate(result)
      }
    ]
  )
end

def final_text(item)
  return unless item.type.to_s == "message" && item.role.to_s == "assistant" &&
                item.status.to_s == "completed" && item.phase.to_s == "final_answer"

  text = item.content.select { |part| part.type.to_s == "output_text" }.map(&:text).join("\n")
  text unless text.empty?
end

def retrieve_answer(sessions, session_id, turn_id)
  page = sessions.turns.items.list(turn_id, session_id: session_id, order: "desc")
  page.auto_paging_each do |item|
    text = final_text(item)
    return text if text && !text.empty?
  end
  nil
end

def prepare_briefing(client, agent_id)
  sessions = client.beta.agents.sessions
  state = {
    session_id: nil,
    stream: nil,
    status: nil,
    answer: nil
  }
  questions = {}
  input_lock = Mutex.new
  cutoff = monotonic_time + 10 * 60

  begin
    Timeout.timeout(12 * 60) do
      now = Time.now.utc
      meeting = (now + 15 * 60).iso8601
      delivery = (now + 12 * 60).iso8601
      stream = sessions.create_streaming(
        agent_id: agent_id,
        environment: { type: "none" },
        input: "I am meeting with Pied Piper (example.com) about our customer-support software " \
          "at #{meeting}. Get me up to speed on their business, recent product " \
          "announcements, and questions I should ask. Send me a concise briefing " \
          "by #{delivery} so I have time to read it before the meeting."
      )
      state[:stream] = stream
      stream.each do |event|
        questions.each_value { |thread| thread.value unless thread.alive? }
        case event
        when OpenAI::Models::Beta::AgentSessionCreatedEvent
          state[:session_id] = event.session.id
          state[:status] = :active
        when OpenAI::Models::Beta::AgentSessionTurnItemDoneEvent
          item = event.item
          if item.is_a?(OpenAI::Models::Beta::AgentFunctionCallItem) &&
             item.name == "request_user_input_async" && !questions.key?(item.call_id)
            questions[item.call_id] = Thread.new(item, event.session_id) do |call, id|
              answer = console_reply(call.arguments, input_lock, cutoff)
              return_result(sessions, id, call, answer)
            rescue OpenAI::Errors::ConflictError
              # The turn can finish while the reply is in flight.
              nil
            end
          end
          state[:answer] = final_text(item) || state[:answer]
        when OpenAI::Models::Beta::AgentSessionTurnCompletedEvent
          state[:status] = :completed
          questions.each_value(&:kill)
          state[:answer] = retrieve_answer(sessions, event.session_id, event.turn.id) || state[:answer]
          break
        when OpenAI::Models::Beta::AgentSessionTurnFailedEvent,
            OpenAI::Models::Beta::AgentSessionTurnCancelledEvent,
            OpenAI::Models::Beta::AgentSessionFailedEvent,
            OpenAI::Models::Beta::AgentSessionEnvironmentFailedEvent
          state[:status] = :failed
          raise "Meeting briefing stopped: #{event.type}"
        when OpenAI::Models::Beta::AgentSessionErrorEvent
          raise "Meeting briefing stopped: #{event.type}"
        end
      end
      raise "The event stream closed before the turn completed." unless state[:status] == :completed
    end
    puts(state[:answer] || "No verified findings were available.")
  rescue Timeout::Error
    puts(state[:answer] || "No verified findings were available by the deadline.")
  ensure
    $stdout.flush
    questions.each_value(&:kill)
    begin
      state[:stream]&.close
      session_id = state[:session_id]
      sessions.events.create(session_id, events: [{ type: "agent.session.input.cancel" }]) if state[:status] == :active && session_id
    ensure
      if state[:session_id]
        # Cancellation settles asynchronously; retry a temporary deletion conflict.
        sessions.delete(
          state[:session_id],
          request_options: {
            max_retries: 5,
            timeout: 10
          }
        )
      end
    end
  end
end

client = OpenAI::Client.new(max_retries: 0, timeout: 12 * 60)
agent = create_agent(client)
begin
  prepare_briefing(client, agent.id)
ensure
  client.beta.agents.delete(agent.id)
end

```


## Time awareness in a Responses API loop

If your personal agent uses a hand-coded [Responses API loop](/api/docs/guides/function-calling), the application needs to keep the model informed of the time left to prepare the briefing. Calculate the deadline once, then append a fresh time reminder before every model request, including requests that submit tool results.

The steps below implement the same meeting-preparation task. Each Responses request includes an updated time reminder for the model. An application timer runs throughout the loop so delivery still happens if a request or tool takes too long.

<details>
<summary>1. Set the meeting and briefing deadlines</summary>

Give the agent the company to research and the time the briefing is due. Keep the same general instructions and [async question tool](/api/docs/guides/async-tool-calling#add-a-tool-to-ask-the-user-for-input), so the model decides whether it needs clarification.

The meeting starts in 15 minutes, the briefing is due in 12, and the application closes unanswered questions 10 minutes after the task starts. Keep those deadlines fixed throughout the task.

Configure the Responses meeting-preparation task

```typescript
import OpenAI from "openai";
import { createInterface } from "node:readline";

const requests = new AbortController();

const instructions =
  "You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. " +
  "Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. " +
  "State important uncertainties.";

const tools: OpenAI.Responses.Tool[] = [
  { type: "web_search" },
  {
    type: "function",
    name: "request_user_input_async",
    description:
      "Ask the user for context. The result contains their reply or reports that the question expired.",
    async: true,
    strict: false,
    parameters: {
      type: "object",
      properties: {
        question: { type: "string" },
        choices: {
          type: "array",
          items: { type: "string" },
          description:
            "Optional suggested replies to display with the question.",
        },
      },
      required: ["question"],
      additionalProperties: false,
    },
  },
];

const startedAt = Date.now();
const monotonicStart = performance.now();
const meetingAt = new Date(startedAt + 15 * 60_000);
const briefingBy = new Date(startedAt + 12 * 60_000);
const deliveryLimit = monotonicStart + 12 * 60_000;
const replyLimit = monotonicStart + 10 * 60_000;
const initialInput: OpenAI.Responses.ResponseInput = [
  {
    role: "user",
    content:
      `I am meeting with Pied Piper (example.com) about our customer-support software at ${meetingAt.toISOString()}. ` +
      "Get me up to speed on their business, recent product announcements, and questions I should ask. " +
      `Send me a concise briefing by ${briefingBy.toISOString()} so I have time to read it before the meeting.`,
  },
];

const responseIds = new Set<string>();
let latestResponseId: string | undefined;
let finalAnswer = "";
let delivered = false;
let deadlineReached = false;
let closed = false;
let replyError: unknown;
```

```python
import asyncio
import json
import os
import sys
from contextlib import suppress
from dataclasses import dataclass, field
from datetime import UTC, datetime, timedelta
from threading import Thread

from openai import APITimeoutError, AsyncOpenAI, OpenAIError
from openai.types.responses import (
    Response,
    ResponseFunctionToolCall,
    ResponseInputItemParam,
    ToolParam,
)

instructions = (
    "You are a personal assistant. Deliver useful, well-sourced answers "
    "within the user's time constraints. Ask for clarification when it "
    "would materially improve the result, and keep making progress while "
    "a reply is pending. State important uncertainties."
)
tools: list[ToolParam] = [
    {"type": "web_search"},
    {
        "type": "function",
        "name": "request_user_input_async",
        "description": "Ask for missing context while continuing independent work.",
        "async": True,
        "strict": False,
        "parameters": {
            "type": "object",
            "properties": {
                "question": {"type": "string"},
                "choices": {"type": "array", "items": {"type": "string"}},
            },
            "required": ["question"],
            "additionalProperties": False,
        },
    },
]

@dataclass
class State:
    response_ids: list[str] = field(default_factory=list)
    questions: dict[str, asyncio.Task[ResponseInputItemParam]] = field(
        default_factory=dict
    )
    seen_calls: set[str] = field(default_factory=set)
    latest_response_id: str | None = None
    saved_answer: str = ""

def configure_task():
    now = datetime.now(UTC)
    started = asyncio.get_running_loop().time()
    meeting = (now + timedelta(minutes=15)).isoformat(timespec="seconds")
    delivery = (now + timedelta(minutes=12)).isoformat(timespec="seconds")
    reply_cutoff = started + 10 * 60
    delivery_cutoff = started + 12 * 60
    input_items = [
        {
            "role": "user",
            "content": (
                f"I am meeting with Pied Piper (example.com) about our customer-support software at "
                f"{meeting}. Get me up to speed on their business, recent product "
                f"announcements, and questions I should ask. Send me a concise briefing by "
                f"{delivery} so I have time to read it before the meeting."
            ),
        }
    ]
    return input_items, reply_cutoff, delivery_cutoff
```

```go
package main

import (
	"bufio"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"strings"
	"sync"
	"time"

	"github.com/openai/openai-go/v3"
	"github.com/openai/openai-go/v3/option"
	"github.com/openai/openai-go/v3/packages/ssestream"
	"github.com/openai/openai-go/v3/responses"
)

const instructions = "You are a personal assistant. Deliver useful, well-sourced answers " +
	"within the user's time constraints. Ask for clarification when it would " +
	"materially improve the result, and keep making progress while a reply is " +
	"pending. State important uncertainties."

func meetingTools() []responses.ToolUnionParam {
	question := responses.ToolParamOfFunction("request_user_input_async", map[string]any{
		"type": "object",
		"properties": map[string]any{
			"question": map[string]string{"type": "string"},
			"choices":  map[string]any{"type": "array", "items": map[string]string{"type": "string"}},
		},
		"required":             []string{"question"},
		"additionalProperties": false,
	}, false)
	question.OfFunction.Async = openai.Bool(true)
	question.OfFunction.Description = openai.String("Ask for missing context while continuing independent work.")
	return []responses.ToolUnionParam{responses.ToolParamOfWebSearch("web_search"), question}
}

func deadlines(now time.Time) (meeting, delivery, replyCutoff time.Time) {
	return now.Add(15 * time.Minute), now.Add(12 * time.Minute), now.Add(10 * time.Minute)
}

func userRequest(meeting, delivery time.Time) responses.ResponseInputParam {
	text := fmt.Sprintf("I am meeting with Pied Piper (example.com) about our customer-support software at %s. "+
		"Get me up to speed on their business, recent product announcements, and questions I should ask. "+
		"Send me a concise briefing by %s so I have time to read it before the meeting.",
		meeting.UTC().Format(time.RFC3339), delivery.UTC().Format(time.RFC3339))
	return responses.ResponseInputParam{responses.ResponseInputItemParamOfMessage(text, "user")}
}
```

```java
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.core.JsonValue;
import com.openai.core.RequestOptions;
import com.openai.core.http.AsyncStreamResponse;
import com.openai.models.responses.EasyInputMessage;
import com.openai.models.responses.FunctionTool;
import com.openai.models.responses.ResponseCreateParams;
import com.openai.models.responses.ResponseFunctionToolCall;
import com.openai.models.responses.ResponseInputItem;
import com.openai.models.responses.ResponseStreamEvent;
import com.openai.models.responses.Tool;
import com.openai.models.responses.WebSearchTool;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicReference;
import java.util.stream.Collectors;

private final String instructions =
    "You are a personal assistant. Deliver useful, well-sourced answers within the user's time"
        + " constraints. Ask for clarification when it would materially improve the result, and"
        + " keep making progress while a reply is pending. State important uncertainties.";
private final List<Tool> tools =
    List.of(
        Tool.ofWebSearch(WebSearchTool.builder().type(WebSearchTool.Type.WEB_SEARCH).build()),
        Tool.ofFunction(
            FunctionTool.builder()
                .name("request_user_input_async")
                .description(
                    "Ask the user for context. The result contains their reply or reports that the question expired.")
                .async(true)
                .strict(false)
                .parameters(
                    FunctionTool.Parameters.builder()
                        .putAdditionalProperty("type", JsonValue.from("object"))
                        .putAdditionalProperty(
                            "properties",
                            JsonValue.from(
                                Map.of(
                                    "question", Map.of("type", "string"),
                                    "choices",
                                        Map.of(
                                            "type",
                                            "array",
                                            "items",
                                            Map.of("type", "string"),
                                            "description",
                                            "Optional suggested replies to display with the question."))))
                        .putAdditionalProperty("required", JsonValue.from(List.of("question")))
                        .putAdditionalProperty("additionalProperties", JsonValue.from(false))
                        .build())
                .build()));
private final Instant startedAt = Instant.now();
private final long monotonicStart = System.nanoTime();
private final Instant meetingAt = startedAt.plus(Duration.ofMinutes(15));
private final Instant briefingBy = startedAt.plus(Duration.ofMinutes(12));
private final long deliveryLimit = monotonicStart + Duration.ofMinutes(12).toNanos();
private final long replyLimit = monotonicStart + Duration.ofMinutes(10).toNanos();
private final List<ResponseInputItem> initialInput =
    List.of(
        ResponseInputItem.ofEasyInputMessage(
            EasyInputMessage.builder()
                .role(EasyInputMessage.Role.USER)
                .content(
                    "I am meeting with Pied Piper (example.com) about our customer-support software at "
                        + meetingAt
                        + ". Get me up to speed on their business, recent product announcements,"
                        + " and questions I should ask. Send me a concise briefing by "
                        + briefingBy
                        + " so I have time to read it before the meeting.")
                .build()));
private final Set<String> responseIds = ConcurrentHashMap.newKeySet();
private final AtomicReference<AsyncStreamResponse<ResponseStreamEvent>> streamRef =
    new AtomicReference<>();
private final AtomicBoolean expired = new AtomicBoolean();
private final AtomicBoolean delivered = new AtomicBoolean();
private final AtomicBoolean closed = new AtomicBoolean();
private final AtomicReference<RuntimeException> replyError = new AtomicReference<>();
private volatile String latestResponseId;
private volatile String finalAnswer = "";
```

```ruby
require "json"
require "openai"
require "time"
require "timeout"

INSTRUCTIONS = "You are a personal assistant. Deliver useful, well-sourced answers " \
  "within the user's time constraints. Ask for clarification when it would " \
  "materially improve the result, and keep making progress while a reply is " \
  "pending. State important uncertainties."
TOOLS = [
  { type: "web_search" },
  {
    type: "function",
    name: "request_user_input_async",
    description: "Ask for missing context while continuing independent work.",
    async: true,
    strict: false,
    parameters: {
      type: "object",
      properties: {
        question: { type: "string" },
        choices: {
          type: "array",
          items: { type: "string" }
        }
      },
      required: ["question"],
      additionalProperties: false
    }
  }
].freeze

def deadlines(now)
  {
    meeting: now + 15 * 60,
    delivery: now + 12 * 60,
    reply: now + 10 * 60
  }
end

def user_request(meeting, delivery)
  {
    role: "user",
    content: "I am meeting with Pied Piper (example.com) about our customer-support software at " \
      "#{meeting.utc.iso8601}. Get me up to speed on their business, recent product " \
      "announcements, and questions I should ask. Send me a concise briefing by " \
      "#{delivery.utc.iso8601} so I have time to read it before the meeting."
  }
end
```

```bash
work=$(mktemp -d)
stream_pid=""
timer_pid=""
reader_pid=""
owned='[]'
latest_response_id=""
pending='{}'
seen='{}'
ready='[]'
exec 4<&0
started=$(date +%s)
reply_cutoff=$((SECONDS + 10 * 60))
delivery_cutoff=$((SECONDS + 12 * 60))
meeting=$(jq -nr --argjson start "$started" '$start + 900 | strftime("%Y-%m-%dT%H:%M:%SZ")')
delivery=$(jq -nr --argjson start "$started" '$start + 720 | strftime("%Y-%m-%dT%H:%M:%SZ")')
input=$(jq -nc --arg meeting "$meeting" --arg delivery "$delivery" '[{
  role: "user",
  content: ("I am meeting with Pied Piper (example.com) about our customer-support software at " +
    $meeting + ". Get me up to speed on their business, recent product announcements, " +
    "and questions I should ask. Send me a concise briefing by " + $delivery +
    " so I have time to read it before the meeting.")
}]')
configuration=$(cat <<'JSON'
{
  "model": "gpt-6.2-astra",
  "instructions": "You are a personal assistant. Deliver useful, well-sourced answers within the user's time constraints. Ask for clarification when it would materially improve the result, and keep making progress while a reply is pending. State important uncertainties.",
  "tools": [
    {"type": "web_search"},
    {
      "type": "function",
      "name": "request_user_input_async",
      "description": "Ask for missing context while continuing independent work.",
      "async": true,
      "strict": false,
      "parameters": {
        "type": "object",
        "properties": {
          "question": {"type": "string"},
          "choices": {"type": "array", "items": {"type": "string"}}
        },
        "required": ["question"],
        "additionalProperties": false
      }
    }
  ],
  "store": true,
  "stream": true
}
JSON
)
```


</details>

<details>
<summary>2. Add the current time before every model request</summary>

The model needs a fresh reading of the clock when it decides what to do next. Use one request helper for both the initial request and every continuation, so the clock update happens in one place.

Immediately before calling the [Responses API](/api/reference/resources/responses/methods/create), append the current UTC time as a [developer message](/api/docs/guides/text#message-roles-and-instruction-following), after the user input or tool results. Send the instructions and tools on every request. The original deadline remains in the conversation history.

Refresh the clock and create a response

```typescript
function request(input: OpenAI.Responses.ResponseInput) {
  return client.responses.create(
    {
      model: "gpt-6.2-astra",
      instructions,
      tools,
      store: true,
      previous_response_id: latestResponseId,
      input: [
        ...input,
        {
          role: "developer",
          content: `Current time: ${new Date().toISOString()}.`,
        },
      ],
      stream: true,
    },
    {
      signal: requests.signal,
      timeout: Math.max(1, Math.ceil(deliveryLimit - performance.now()) + 1000),
    }
  );
}
```

```python
async def start_response(input_items, previous_response_id, delivery_cutoff):
    remaining = delivery_cutoff - asyncio.get_running_loop().time()
    if remaining <= 0:
        raise TimeoutError
    current_time = datetime.now(UTC).isoformat(timespec="seconds")
    return await client.responses.create(
        model="gpt-6.2-astra",
        instructions=instructions,
        tools=tools,
        previous_response_id=previous_response_id,
        input=[
            *input_items,
            {"role": "developer", "content": f"Current time: {current_time}."},
        ],
        store=True,
        stream=True,
        timeout=remaining,
    )
```

```go
func startResponse(ctx context.Context, client *openai.Client, input responses.ResponseInputParam, previousID string) *ssestream.Stream[responses.ResponseStreamEventUnion] {
	clock := responses.ResponseInputItemParamOfMessage(
		"Current time: "+time.Now().UTC().Format(time.RFC3339)+".", "developer")
	params := responses.ResponseNewParams{
		Model:        "gpt-6.2-astra",
		Tools:        meetingTools(),
		Instructions: openai.String(instructions),
		Store:        openai.Bool(true),
		Input:        responses.ResponseNewParamsInputUnion{OfInputItemList: append(input, clock)},
	}
	if previousID != "" {
		params.PreviousResponseID = openai.String(previousID)
	}
	return client.Responses.NewStreaming(ctx, params)
}
```

```java
private AsyncStreamResponse<ResponseStreamEvent> request(List<ResponseInputItem> input) {
  var params =
      ResponseCreateParams.builder()
          .model("gpt-6.2-astra")
          .instructions(instructions)
          .tools(tools)
          .store(true)
          .previousResponseId(Optional.ofNullable(latestResponseId));
  var options =
      RequestOptions.builder()
          .timeout(
              Duration.ofNanos(Math.max(1, deliveryLimit - System.nanoTime())).plusSeconds(1))
          .build();
  var requestInput = new ArrayList<>(input);
  requestInput.add(
      ResponseInputItem.ofEasyInputMessage(
          EasyInputMessage.builder()
              .role(EasyInputMessage.Role.DEVELOPER)
              .content("Current time: " + Instant.now() + ".")
              .build()));
  return client
      .async()
      .responses()
      .createStreaming(params.inputOfResponse(requestInput).build(), options);
}
```

```ruby
def start_response(client, input, previous_id)
  reminder = {
    role: "developer",
    content: "Current time: #{Time.now.utc.iso8601}."
  }
  params = {
    model: "gpt-6.2-astra",
    tools: TOOLS,
    instructions: INSTRUCTIONS,
    store: true,
    input: [*input, reminder]
  }
  params[:previous_response_id] = previous_id if previous_id
  client.responses.stream_raw(params)
end
```

```bash
start_response() {
  local remaining current_time request
  remaining=$((delivery_cutoff - SECONDS))
  if [ "$remaining" -le 0 ]; then deliver; exit 0; fi
  rm -f "$work/events"
  mkfifo "$work/events"
  current_time=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  request=$(jq -nc --argjson config "$configuration" --argjson input "$input" \
    --arg previous "$latest_response_id" --arg now "$current_time" '
    $config + {
      input: ($input + [{role: "developer", content: ("Current time: " + $now + ".")}])
    } + (if $previous == "" then {} else {previous_response_id: $previous} end)')
  curl --fail-with-body --silent --show-error --no-buffer --max-time "$remaining" \
    https://api.openai.com/v1/responses \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    --data-binary "$request" >"$work/events" &
  stream_pid=$!
  exec 3<"$work/events"
}
```


</details>

<details>
<summary>3. Collect a reply while research continues</summary>

When the agent asks about the meeting’s focus, display the question and collect the reply independently of response generation. The collector returns the answer when it arrives or `no_answer` at the cutoff, using the original `call_id` to keep the result attached to the right question.

Collect an asynchronous clarification result

```typescript
const terminal = createInterface({ input: process.stdin });
const bufferedReplies: string[] = [];
const pendingReplies = new Map<string, (answer?: string) => void>();
const seenCalls = new Set<string>();
const replyTasks = new Set<Promise<void>>();
const readyResults: OpenAI.Responses.ResponseInput = [];

terminal.on("line", (line) => {
  const waiting = pendingReplies.values().next().value;
  if (waiting) {
    waiting(
      performance.now() < replyLimit ? line.trim() || undefined : undefined
    );
  } else bufferedReplies.push(line.trim());
});

function readReply(callId: string): Promise<string | undefined> {
  return new Promise((resolve) => {
    let timer: ReturnType<typeof setTimeout> | undefined;
    const finish = (answer?: string) => {
      clearTimeout(timer);
      pendingReplies.delete(callId);
      resolve(answer);
    };
    pendingReplies.set(callId, finish);
    timer = setTimeout(finish, Math.max(0, replyLimit - performance.now()));
    if (performance.now() >= replyLimit) finish();
    else if (bufferedReplies.length)
      finish(bufferedReplies.shift() || undefined);
  });
}

function closeQuestions() {
  closed = true;
  for (const finish of pendingReplies.values()) finish();
  terminal.close();
  process.stdin.pause();
}

async function collectReply(call: OpenAI.Responses.ResponseFunctionToolCall) {
  const { question, choices } = JSON.parse(call.arguments);
  console.error(question);
  if (choices?.length) console.error(choices.join(" / "));
  const answer = await readReply(call.call_id);
  if (closed) return;
  readyResults.push({
    type: "function_call_output",
    call_id: call.call_id,
    output: JSON.stringify(
      answer ? { status: "answered", answer } : { status: "no_answer" }
    ),
  });
}
```

```python
def console_replies() -> asyncio.Queue[str]:
    loop = asyncio.get_running_loop()
    replies: asyncio.Queue[str] = asyncio.Queue()

    def read_lines():
        with os.fdopen(
            os.dup(sys.stdin.fileno()), encoding=sys.stdin.encoding
        ) as terminal:
            for line in terminal:
                try:
                    loop.call_soon_threadsafe(replies.put_nowait, line.strip())
                except RuntimeError:
                    return

    Thread(target=read_lines, daemon=True).start()
    return replies

async def collect_reply(call, replies, prompt_lock, reply_cutoff):
    arguments = json.loads(call.arguments)
    answer = ""
    # An expired question keeps the empty answer and returns no_answer.
    with suppress(TimeoutError):
        async with asyncio.timeout_at(reply_cutoff), prompt_lock:
            if asyncio.get_running_loop().time() >= reply_cutoff:
                raise TimeoutError
            print(arguments["question"], file=sys.stderr, flush=True)
            for choice in arguments.get("choices", []):
                print(f"  {choice}", file=sys.stderr, flush=True)
            reply = await replies.get()
            if asyncio.get_running_loop().time() < reply_cutoff:
                answer = reply
    result = (
        {"status": "answered", "answer": answer} if answer else {"status": "no_answer"}
    )
    return {
        "type": "function_call_output",
        "call_id": call.call_id,
        "output": json.dumps(result),
    }
```

```go
func consoleReplies(ctx context.Context) <-chan string {
	lines := make(chan string)
	go func() {
		// Leave the channel open on EOF. The reply cutoff resolves unanswered questions.
		scanner := bufio.NewScanner(os.Stdin)
		for scanner.Scan() {
			select {
			case lines <- strings.TrimSpace(scanner.Text()):
			case <-ctx.Done():
				return
			}
		}
	}()
	return lines
}

type questionBox struct {
	lines      <-chan string
	cutoff     time.Time
	promptLock sync.Mutex
	jobs       map[string]<-chan string
}

func (box *questionBox) ask(ctx context.Context, call responses.ResponseFunctionToolCall) error {
	if _, seen := box.jobs[call.CallID]; seen {
		return nil
	}
	var args struct {
		Question string
		Choices  []string
	}
	if err := json.Unmarshal([]byte(call.Arguments), &args); err != nil {
		return err
	}
	if args.Question == "" {
		return errors.New("missing clarification question")
	}
	result := make(chan string, 1)
	box.jobs[call.CallID] = result
	go func() {
		box.promptLock.Lock()
		defer box.promptLock.Unlock()
		replyCtx, cancel := context.WithDeadline(ctx, box.cutoff)
		defer cancel()
		answer := ""
		if replyCtx.Err() == nil {
			fmt.Fprintln(os.Stderr, args.Question)
			for _, choice := range args.Choices {
				fmt.Fprintln(os.Stderr, " ", choice)
			}
			select {
			case line := <-box.lines:
				if time.Now().Before(box.cutoff) {
					answer = line
				}
			case <-replyCtx.Done():
			}
		}
		result <- answer
	}()
	return nil
}
```

```java
public record Question(String question, List<String> choices) {}

private final BlockingQueue<String> replies = new LinkedBlockingQueue<>();
private final BlockingQueue<ResponseInputItem> readyResults = new LinkedBlockingQueue<>();
private final Set<String> seenCalls = ConcurrentHashMap.newKeySet();
private final ExecutorService replyWorker = Executors.newSingleThreadExecutor();

private void collectReply(ResponseFunctionToolCall call) {
  try {
    var question = call.arguments(Question.class);
    System.err.println(question.question());
    if (question.choices() != null) System.err.println(String.join(" / ", question.choices()));
    long remaining = Math.max(0, replyLimit - System.nanoTime());
    String answer = remaining == 0 ? null : replies.poll(remaining, TimeUnit.NANOSECONDS);
    if (System.nanoTime() >= replyLimit) answer = null;
    if (closed.get()) return;
    var result =
        answer == null || answer.isBlank()
            ? Map.of("status", "no_answer")
            : Map.of("status", "answered", "answer", answer);
    readyResults.add(
        ResponseInputItem.ofFunctionCallOutput(
            ResponseInputItem.FunctionCallOutput.builder()
                .callId(call.callId())
                .output(new JsonMapper().writeValueAsString(result))
                .build()));
  } catch (InterruptedException ignored) {
    Thread.currentThread().interrupt();
  } catch (Exception error) {
    if (!closed.get()) {
      replyError.set(new IllegalStateException("Could not collect the reply", error));
      closeStream();
    }
  }
}

private void startInputReader() {
  var input =
      new Thread(
          () -> {
            try {
              var reader = new BufferedReader(new InputStreamReader(System.in));
              String line;
              while (!closed.get() && (line = reader.readLine()) != null)
                replies.add(line.trim());
            } catch (java.io.IOException error) {
              System.err.println("Input closed: " + error.getMessage());
            }
          });
  input.setDaemon(true);
  input.start();
}

private void closeQuestions() {
  closed.set(true);
  replyWorker.shutdownNow();
}
```

```ruby
def read_answer(arguments, input_lock, cutoff)
  remaining = cutoff - Time.now
  return if remaining <= 0

  Timeout.timeout(remaining) do
    input_lock.synchronize do
      return if Time.now >= cutoff

      warn arguments.fetch("question")
      Array(arguments["choices"]).each { |choice| warn "  #{choice}" }
      line = $stdin.gets
      sleep([cutoff - Time.now, 0].max) if line.nil?
      line&.strip if Time.now < cutoff
    end
  end
rescue Timeout::Error
  nil
end

def start_question(call, input_lock, cutoff)
  Thread.new do
    answer = read_answer(JSON.parse(call.arguments), input_lock, cutoff)
    result = { status: "no_answer" }
    unless answer.nil? || answer.empty?
      result = {
        status: "answered",
        answer: answer
      }
    end
    {
      type: "function_call_output",
      call_id: call.call_id,
      output: JSON.generate(result)
    }
  end
end
```

```bash
# Buffer terminal replies independently of the event stream.
next_reply=1
(
  number=1
  while IFS= read -r line || [ -n "$line" ]; do
    printf '%s\n' "$line" >"$work/incoming"
    mv "$work/incoming" "$work/reply-$number"
    number=$((number + 1))
  done
) <&4 &
reader_pid=$!

register_question() {
  local call=$1 call_id
  call_id=$(jq -r '.call_id' <<<"$call")
  if jq -e --arg id "$call_id" 'has($id)' <<<"$seen" >/dev/null; then return; fi
  seen=$(jq -c --arg id "$call_id" '.[$id] = true' <<<"$seen")
  pending=$(jq -c --arg id "$call_id" --argjson call "$call" '.[$id] = $call' <<<"$pending")
  if [ "$SECONDS" -lt "$reply_cutoff" ]; then
    jq -r '.arguments | fromjson | .question, (.choices[]? // empty)' <<<"$call" >&2
  fi
}

collect_replies() {
  local call_id answer result
  while :; do
    call_id=$(jq -r 'keys_unsorted[0] // empty' <<<"$pending")
    if [ -z "$call_id" ]; then break; fi
    answer=""
    if [ "$SECONDS" -lt "$reply_cutoff" ]; then
      if [ ! -f "$work/reply-$next_reply" ]; then break; fi
      IFS= read -r answer <"$work/reply-$next_reply"
      next_reply=$((next_reply + 1))
    fi
    if [ "$SECONDS" -ge "$reply_cutoff" ]; then answer=""; fi
    result=$(jq -nc --arg answer "$answer" '
      if $answer == "" then {status: "no_answer"}
      else {status: "answered", answer: $answer} end')
    ready=$(jq -c --arg id "$call_id" --arg result "$result" '. + [{
      type: "function_call_output", call_id: $id, output: $result
    }]' <<<"$ready")
    pending=$(jq -c --arg id "$call_id" 'del(.[$id])' <<<"$pending")
  done
}
```


</details>

<details>
<summary>4. Handle questions and completed responses</summary>

Start the input collector as soon as the complete question arrives in the [response stream](/api/docs/guides/streaming-responses). Keep reading events while the user considers it, so the model can continue its web research.

When a response completes, save its ID for the next request and retain any completed briefing for delivery. The event handler also records response IDs for cleanup.

Read streamed questions and completed answers

```typescript
async function consumeEvents(
  stream: AsyncIterable<OpenAI.Responses.ResponseStreamEvent>
) {
  for await (const event of stream) {
    if (event.type === "response.created") responseIds.add(event.response.id);
    if (
      event.type === "response.output_item.done" &&
      event.item.type === "function_call" &&
      event.item.name === "request_user_input_async" &&
      !seenCalls.has(event.item.call_id)
    ) {
      seenCalls.add(event.item.call_id);
      // Continue consuming events while the terminal waits for a reply.
      const task = collectReply(event.item).catch((error: unknown) => {
        replyError = error;
        requests.abort();
      });
      replyTasks.add(task);
    }
    if (event.type === "response.completed") {
      latestResponseId = event.response.id;
      const answer = event.response.output
        .filter(
          (item) =>
            item.type === "message" &&
            item.status === "completed" &&
            item.phase !== "commentary"
        )
        .flatMap((item) => (item.type === "message" ? item.content : []))
        .filter((part) => part.type === "output_text")
        .map((part) => part.text)
        .join("\n");
      if (answer) finalAnswer = answer;
      return answer;
    }
    if (
      event.type === "response.failed" ||
      event.type === "response.incomplete"
    ) {
      throw new Error(`Response did not complete: ${event.type}`);
    }
    if (event.type === "error") throw new Error(event.message);
  }
  throw new Error("The stream closed before response completion");
}
```

```python
def final_text(response: Response) -> str:
    return "\n".join(
        part.text
        for item in response.output
        if item.type == "message"
        and item.status == "completed"
        and item.phase in (None, "final_answer")
        for part in item.content
        if part.type == "output_text"
    )

async def read_response(stream, state, replies, prompt_lock, reply_cutoff):
    async with stream:
        async for event in stream:
            if event.type == "response.created":
                state.response_ids.append(event.response.id)
            elif event.type == "response.output_item.done":
                call = event.item
                if (
                    isinstance(call, ResponseFunctionToolCall)
                    and call.name == "request_user_input_async"
                    and call.call_id not in state.seen_calls
                ):
                    state.seen_calls.add(call.call_id)
                    state.questions[call.call_id] = asyncio.create_task(
                        collect_reply(call, replies, prompt_lock, reply_cutoff)
                    )
            elif event.type == "response.completed":
                state.latest_response_id = event.response.id
                answer = final_text(event.response)
                state.saved_answer = answer or state.saved_answer
                # Give the transport a moment to finish after the terminal event.
                with suppress(TimeoutError):
                    async with asyncio.timeout(1):
                        async for _ in stream:
                            continue
                return answer
            elif event.type in ("response.failed", "response.incomplete", "error"):
                raise RuntimeError(f"Meeting briefing stopped: {event.type}")
    raise RuntimeError("The response stream closed before completion.")
```

```go
func finalText(response responses.Response) string {
	var parts []string
	for _, item := range response.Output {
		if item.Type != "message" {
			continue
		}
		message := item.AsMessage()
		if message.Role != "assistant" || message.Status != "completed" || (message.Phase != "" && message.Phase != "final_answer") {
			continue
		}
		for _, part := range message.Content {
			if part.Type == "output_text" {
				parts = append(parts, part.Text)
			}
		}
	}
	return strings.Join(parts, "\n")
}

type progress struct {
	latestID, savedAnswer string
	responseIDs           map[string]bool
}

func (s *progress) consume(ctx context.Context, stream *ssestream.Stream[responses.ResponseStreamEventUnion], box *questionBox) (string, error) {
	defer stream.Close()
	for stream.Next() {
		event := stream.Current()
		switch event.Type {
		case "response.created":
			s.responseIDs[event.Response.ID] = true
		case "response.output_item.done":
			if event.Item.Type == "function_call" {
				call := event.Item.AsFunctionCall()
				if call.Name == "request_user_input_async" {
					if err := box.ask(ctx, call); err != nil {
						return "", err
					}
				}
			}
		case "response.completed":
			s.latestID = event.Response.ID
			s.responseIDs[s.latestID] = true
			answer := finalText(event.Response)
			if answer != "" {
				s.savedAnswer = answer
			}
			return answer, nil
		case "response.failed", "response.incomplete", "error":
			return "", fmt.Errorf("response stopped: %s", event.Type)
		}
	}
	if err := stream.Err(); err != nil {
		return "", err
	}
	return "", errors.New("stream closed before response.completed")
}
```

```java
private String consumeEvents(AsyncStreamResponse<ResponseStreamEvent> stream) {
  var answerReady = new CompletableFuture<String>();
  stream
      .onCompleteFuture()
      .whenComplete(
          (ignored, error) -> {
            if (!answerReady.isDone()) {
              answerReady.completeExceptionally(
                  error != null
                      ? error
                      : new IllegalStateException(
                          "The stream closed before response completion"));
            }
          });
  stream.subscribe(
      new AsyncStreamResponse.Handler<>() {
        @Override
        public void onNext(ResponseStreamEvent event) {
          event.created().ifPresent(created -> responseIds.add(created.response().id()));
          event
              .outputItemDone()
              .flatMap(done -> done.item().functionCall())
              .ifPresent(
                  call -> {
                    if (call.name().equals("request_user_input_async")
                        && seenCalls.add(call.callId())) {
                      // Collect the reply separately while this handler consumes events.
                      replyWorker.submit(() -> collectReply(call));
                    }
                  });
          event
              .completed()
              .ifPresent(
                  completed -> {
                    latestResponseId = completed.response().id();
                    String answer =
                        completed.response().output().stream()
                            .flatMap(item -> item.message().stream())
                            .filter(message -> message.status().toString().equals("completed"))
                            .filter(
                                message ->
                                    !message
                                        .phase()
                                        .map(phase -> phase.toString().equals("commentary"))
                                        .orElse(false))
                            .flatMap(message -> message.content().stream())
                            .flatMap(content -> content.outputText().stream())
                            .map(text -> text.text())
                            .collect(Collectors.joining("\n"));
                    if (!answer.isBlank()) finalAnswer = answer;
                    answerReady.complete(answer);
                  });
          if (event.failed().isPresent()
              || event.incomplete().isPresent()
              || event.error().isPresent()) {
            answerReady.completeExceptionally(
                new IllegalStateException("Response did not complete: " + event));
          }
        }

        @Override
        public void onComplete(Optional<Throwable> error) {
          if (!answerReady.isDone()) {
            answerReady.completeExceptionally(
                error.orElseGet(
                    () ->
                        new IllegalStateException(
                            "The stream closed before response completion")));
          }
        }
      });
  return answerReady.join();
}
```

```ruby
def final_text(item)
  return unless item.is_a?(OpenAI::Models::Responses::ResponseOutputMessage)
  return unless item.role.to_s == "assistant" && item.status.to_s == "completed"
  return unless item.phase.nil? || item.phase.to_s == "final_answer"

  text = item.content.filter_map { |part| part.text if part.is_a?(OpenAI::Models::Responses::ResponseOutputText) }.join("\n")
  text unless text.empty?
end

def consume(stream, state, jobs, seen, input_lock, cutoff)
  stream.each do |event|
    case event
    when OpenAI::Models::Responses::ResponseCreatedEvent
      state[:ids][event.response.id] = true
    when OpenAI::Models::Responses::ResponseOutputItemDoneEvent
      item = event.item
      if item.is_a?(OpenAI::Models::Responses::ResponseFunctionToolCall) &&
         item.name == "request_user_input_async" && !seen.key?(item.call_id)
        seen[item.call_id] = true
        jobs[item.call_id] = start_question(item, input_lock, cutoff)
      end
    when OpenAI::Models::Responses::ResponseCompletedEvent
      state[:ids][event.response.id] = true
      state[:latest_id] = event.response.id
      answer = event.response.output.filter_map { |item| final_text(item) }.join("\n")
      state[:saved_answer] = answer unless answer.empty?
      return answer
    when OpenAI::Models::Responses::ResponseFailedEvent,
        OpenAI::Models::Responses::ResponseIncompleteEvent,
        OpenAI::Models::Responses::ResponseErrorEvent
      raise "Response stopped: #{event.type}"
    end
  end
  raise "The stream closed before response.completed."
end
```

```bash
read_response() {
  local chunk buffer="" line event event_type id
  current_answer=""
  while :; do
    chunk=""
    if IFS= read -r -t 1 -u 3 chunk; then
      line="$buffer$chunk"
      buffer=""
      if [[ "$line" == 'data: '* && "$line" != 'data: [DONE]' ]]; then
        event=${line#data: }
        event_type=$(jq -r '.type' <<<"$event")
        if [ "$event_type" = response.created ]; then
          id=$(jq -er '.response.id' <<<"$event")
          owned=$(jq -c --arg id "$id" 'if index($id) then . else . + [$id] end' <<<"$owned")
        elif [ "$event_type" = response.output_item.done ] && \
          jq -e '.item.type == "function_call" and .item.name == "request_user_input_async"' \
            <<<"$event" >/dev/null; then
          register_question "$(jq -c '.item' <<<"$event")"
        elif [ "$event_type" = response.completed ]; then
          latest_response_id=$(jq -er '.response.id' <<<"$event")
          current_answer=$(jq -r '[.response.output[] |
            select(.type == "message" and .status == "completed" and
              (.phase == "final_answer" or .phase == null)) |
            .content[] | select(.type == "output_text") | .text] | join("\n")' <<<"$event")
          if [ -n "$current_answer" ]; then
            printf '%s\n' "$current_answer" >"$work/candidate.txt"
            mv "$work/candidate.txt" "$work/final.txt"
          fi
          collect_replies
          if wait "$stream_pid"; then :
          else
            status=$?
            if [ "$status" -eq 28 ]; then deliver; exit 0; fi
            return "$status"
          fi
          stream_pid=""
          exec 3<&-
          return
        elif [[ "$event_type" == response.failed || "$event_type" == response.incomplete || "$event_type" == error ]]; then
          printf 'Meeting briefing stopped: %s\n' "$event_type" >&2
          return 1
        fi
      fi
    else
      buffer="$buffer$chunk"
      if ! kill -0 "$stream_pid" 2>/dev/null; then
        if wait "$stream_pid"; then :
        else
          status=$?
          if [ "$status" -eq 28 ]; then deliver; exit 0; fi
        fi
        printf '%s\n' 'The response stream closed before completion.' >&2
        return 1
      fi
    fi
    collect_replies
  done
}
```


</details>

<details>
<summary>5. Continue the loop with ready results</summary>

A reply that arrives during research should inform the next version of the briefing. After each response finishes, collect any ready tool results and send them through the request helper, which adds a new time reminder. Use the latest completed response ID as [`previous_response_id`](/api/docs/guides/conversation-state#passing-context-from-the-previous-response), even when a result belongs to a question from an earlier response.

Keep requests sequential so each builds on the latest conversation state. If the briefing is complete and no result is ready, deliver it and close pending questions. Otherwise, continue working with the updated time, without waiting for an answer.

Continue research and incorporate replies

```typescript
async function runLoop() {
  let nextInput = initialInput;
  while (!delivered) {
    const answer = await consumeEvents(await request(nextInput));
    if (replyError) throw replyError;
    nextInput = readyResults.splice(0);
    if (nextInput.length > 0) continue;
    if (answer) {
      closeQuestions();
      deliver(answer);
    }
    // With no final answer or ready result, the next request continues work.
    // It receives a fresh clock message without waiting for pending replies.
  }
}
```

```python
async def run_loop(input_items, state, replies, reply_cutoff, delivery_cutoff):
    prompt_lock = asyncio.Lock()
    while True:
        stream = await start_response(
            input_items, state.latest_response_id, delivery_cutoff
        )
        answer = await read_response(stream, state, replies, prompt_lock, reply_cutoff)
        await asyncio.sleep(0)  # Let ready input handlers finish.
        input_items = []
        for call_id, task in list(state.questions.items()):
            if task.done():
                input_items.append(task.result())
                del state.questions[call_id]
        if input_items:
            continue  # Incorporate replies that arrived during generation.
        if answer:
            return answer
        input_items = [
            {
                "role": "developer",
                "content": "Continue working on the briefing while replies are pending.",
            }
        ]
```

```go
func (box *questionBox) ready() (responses.ResponseInputParam, error) {
	var input responses.ResponseInputParam
	for callID, job := range box.jobs {
		if job == nil {
			continue
		}
		select {
		case answer := <-job:
			result := map[string]string{"status": "no_answer"}
			if answer != "" {
				result["status"], result["answer"] = "answered", answer
			}
			output, err := json.Marshal(result)
			if err != nil {
				return nil, err
			}
			item := responses.ResponseInputItemParamOfFunctionCallOutput(string(output))
			item.OfFunctionCallOutput.CallID = openai.String(callID)
			input = append(input, item)
			box.jobs[callID] = nil // Keep the ID so a replay cannot submit it twice.
		default:
		}
	}
	return input, nil
}

func (s *progress) prepare(ctx context.Context, client *openai.Client, input responses.ResponseInputParam, box *questionBox) (string, error) {
	for ctx.Err() == nil {
		stream := startResponse(ctx, client, input, s.latestID)
		answer, err := s.consume(ctx, stream, box)
		if err != nil {
			return "", err
		}
		next, err := box.ready()
		if err != nil {
			return "", err
		}
		if len(next) > 0 {
			input = next
			continue
		}
		if answer != "" {
			return answer, nil
		}
		input = responses.ResponseInputParam{responses.ResponseInputItemParamOfMessage(
			"Continue preparing the briefing using the context available.", "developer")}
	}
	return "", ctx.Err()
}
```

```java
private void runLoop() {
  List<ResponseInputItem> nextInput = initialInput;
  while (!delivered.get()) {
    String answer;
    var stream = request(nextInput);
    try {
      streamRef.set(stream);
      if (expired.get()) return;
      answer = consumeEvents(stream);
    } finally {
      streamRef.set(null);
      stream.close();
    }
    if (replyError.get() != null) throw replyError.get();
    nextInput = new ArrayList<>();
    readyResults.drainTo(nextInput);
    if (!nextInput.isEmpty()) continue;
    if (!answer.isBlank()) {
      closeQuestions();
      deliver(answer);
    }
    // Otherwise continue work with a fresh clock message, without waiting for input.
  }
}
```

```ruby
def prepare(client, state, jobs, times)
  input = [user_request(times[:meeting], times[:delivery])]
  seen = {}
  input_lock = Mutex.new
  loop do
    stream = start_response(client, input, state[:latest_id])
    begin
      answer = consume(stream, state, jobs, seen, input_lock, times[:reply])
    ensure
      stream.close
    end
    ready = jobs.keys.reject { |call_id| jobs.fetch(call_id).alive? }
    input.replace(ready.map { |call_id| jobs.delete(call_id).value })
    next unless input.empty?
    return answer unless answer.empty?

    input << {
      role: "developer",
      content: "Continue preparing the briefing using the context available."
    }
  end
end
```

```bash
while [ "$SECONDS" -lt "$delivery_cutoff" ]; do
  start_response
  read_response
  collect_replies
  if [ "$(jq length <<<"$ready")" -gt 0 ]; then
    input="$ready"
    ready='[]'
    continue
  fi
  if [ -n "$current_answer" ]; then deliver; exit 0; fi
  input='[{"role":"developer","content":"Continue working on the briefing while replies are pending."}]'
done
deliver
```


</details>

<details>
<summary>6. Enforce delivery while a request is in flight</summary>

An application timer enforces the briefing deadline, preserving the user’s review time. Apply the timer to the whole loop, including API requests and stream consumption.

At the deadline, stop the active request and deliver the saved briefing, or report that no verified findings are ready. Close pending input handlers and [delete the stored responses](/api/reference/resources/responses/methods/delete) after delivery.

Deliver on time and release task resources

```typescript
function deliver(text: string) {
  if (delivered) return;
  delivered = true;
  console.log(text);
}

const deliveryTimer = setTimeout(
  () => {
    deadlineReached = true;
    deliver(
      finalAnswer || "No results were verified by the briefing deadline."
    );
    closeQuestions();
    requests.abort();
  },
  Math.max(0, deliveryLimit - performance.now())
);

try {
  await runLoop();
} catch (error) {
  if (!deadlineReached) throw replyError || error;
} finally {
  clearTimeout(deliveryTimer);
  closeQuestions();
  requests.abort();
  await Promise.allSettled(replyTasks);
  const cleanup = await Promise.allSettled(
    [...responseIds].map((id) => client.responses.delete(id))
  );
  const failures = cleanup.filter((result) => result.status === "rejected");
  if (failures.length) {
    console.error("Could not delete every stored response", failures);
    process.exitCode = 1;
  }
}
```

```python
async def main():
    state = State()
    replies = console_replies()
    input_items, reply_cutoff, delivery_cutoff = configure_task()
    async with client:
        try:
            async with asyncio.timeout_at(delivery_cutoff):
                answer = await run_loop(
                    input_items, state, replies, reply_cutoff, delivery_cutoff
                )
            print(answer, flush=True)
        except (TimeoutError, APITimeoutError):
            print(
                state.saved_answer
                or "No verified findings were available by the deadline.",
                flush=True,
            )
        finally:
            for task in state.questions.values():
                task.cancel()
            await asyncio.gather(*state.questions.values(), return_exceptions=True)
            for response_id in reversed(state.response_ids):
                try:
                    await client.responses.delete(response_id, timeout=10)
                except OpenAIError as error:
                    print(f"Could not delete {response_id}: {error}", file=sys.stderr)


if __name__ == "__main__":
    asyncio.run(main())
```

```go
func run(client *openai.Client) error {
	meeting, delivery, cutoff := deadlines(time.Now())
	ctx, cancel := context.WithDeadline(context.Background(), delivery)
	defer cancel()
	state := progress{responseIDs: map[string]bool{}}
	box := questionBox{lines: consoleReplies(ctx), cutoff: cutoff, jobs: map[string]<-chan string{}}
	answer, err := state.prepare(ctx, client, userRequest(meeting, delivery), &box)
	if errors.Is(ctx.Err(), context.DeadlineExceeded) {
		answer, err = state.savedAnswer, nil
		if answer == "" {
			answer = "No verified findings were available by the deadline."
		}
	}
	if err == nil {
		fmt.Println(answer)
	}
	cancel()
	os.Stdin.Close()
	for id := range state.responseIDs {
		cleanup, stop := context.WithTimeout(context.Background(), 10*time.Second)
		deleteErr := client.Responses.Delete(cleanup, id)
		stop()
		err = errors.Join(err, deleteErr)
	}
	return err
}
```

```java
private final ScheduledExecutorService timers = Executors.newSingleThreadScheduledExecutor();

private void deliver(String text) {
  if (delivered.compareAndSet(false, true)) System.out.println(text);
}

private void closeStream() {
  var stream = streamRef.getAndSet(null);
  if (stream != null) stream.close();
}

private void run() {
  startInputReader();
  var deliveryTimer =
      timers.schedule(
          () -> {
            expired.set(true);
            deliver(
                finalAnswer.isBlank()
                    ? "No results were verified by the briefing deadline."
                    : finalAnswer);
            closeQuestions();
            closeStream();
          },
          Math.max(0, deliveryLimit - System.nanoTime()),
          TimeUnit.NANOSECONDS);
  try {
    runLoop();
  } catch (RuntimeException error) {
    if (!expired.get()) throw error;
  } finally {
    deliveryTimer.cancel(false);
    closeQuestions();
    closeStream();
    timers.shutdownNow();
    RuntimeException cleanupFailure = null;
    for (String id : responseIds) {
      try {
        client.responses().delete(id);
      } catch (RuntimeException error) {
        if (cleanupFailure == null)
          cleanupFailure = new IllegalStateException("Could not delete every stored response");
        cleanupFailure.addSuppressed(error);
      }
    }
    client.close();
    if (cleanupFailure != null) throw cleanupFailure;
  }
}
```

```ruby
times = deadlines(Time.now.utc)
state = {
  ids: {},
  latest_id: nil,
  saved_answer: nil
}
jobs = {}
begin
  answer = Timeout.timeout(times[:delivery] - Time.now) do
    prepare(client, state, jobs, times)
  end
  puts answer
rescue Timeout::Error
  puts(state[:saved_answer] || "No verified findings were available by the deadline.")
ensure
  $stdout.flush
  jobs.each_value(&:kill)
  state[:ids].each_key do |id|
    client.responses.delete(id)
  rescue OpenAI::Errors::APIError => error
    warn "Could not delete response #{id}: #{error.message}"
  end
end
```

```bash
cleanup() {
  local exit_status=$?
  trap - EXIT TERM
  for pid in "$stream_pid" "$timer_pid" "$reader_pid"; do
    if [ -n "$pid" ]; then
      kill "$pid" 2>/dev/null || true
      wait "$pid" 2>/dev/null || true
    fi
  done
  while IFS= read -r response_id; do
    curl --fail-with-body --silent --show-error --max-time 10 -X DELETE \
      "https://api.openai.com/v1/responses/$response_id" \
      -H "Authorization: Bearer $OPENAI_API_KEY" >/dev/null || exit_status=1
  done < <(jq -r 'reverse[]' <<<"$owned")
  rm -rf "$work"
  exit "$exit_status"
}

deliver() {
  if mkdir "$work/delivered" 2>/dev/null; then
    if [ -s "$work/final.txt" ]; then cat "$work/final.txt"
    else printf '%s\n' 'No verified findings were available by the deadline.'
    fi
  fi
}

trap cleanup EXIT
trap 'exit 0' TERM
(
  sleep "$((delivery_cutoff - SECONDS))" &
  sleep_pid=$!
  trap 'kill "$sleep_pid" 2>/dev/null || true; wait "$sleep_pid" 2>/dev/null || true; exit 0' TERM
  wait "$sleep_pid"
  deliver
  kill -TERM "$$"
) &
timer_pid=$!
```


</details>

## Conclusion

Personal agents sometimes need to complete work by a deadline. Ideally they ask for clarification to improve the result, but even without that clarification, they still need to perform useful work.

We gave the agent a deadline and enabled time awareness to help it decide whether to investigate further, verify a finding, or finish writing. Async function tools let it keep researching while the user considered its question. The agent still has enough context to write a useful meeting briefing if the question goes unanswered.

ChatGPT and other agent harnesses use this pattern to keep working while asking the user for clarification. We hope that including time awareness in the Agents API helps you build this pattern yourself. [Let us know what you build](https://x.com/OpenAIDevs)!