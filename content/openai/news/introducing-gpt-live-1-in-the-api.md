Title: Build more natural voice experiences with GPT‑Live‑1 in the API

URL Source: https://openai.com/index/introducing-gpt-live-1-in-the-api

Markdown Content:
We’re launching GPT‑Live‑1 in the API, giving developers a powerful, natural voice model for building voice-enabled apps and business workflows. [__First introduced in ChatGPT__](https://openai.com/index/introducing-gpt-live/), GPT‑Live‑1 is capable of listening and speaking at the same time, and, as [__seen with Codex and ChatGPT Work__(opens in a new window)](https://www.youtube.com/watch?v=E0ZMOschrTU), can delegate deeper reasoning and actions to the models and tools it is paired with.

For the API release of GPT‑Live‑1, we’ve focused on new capabilities that let developers steer and customize voice experiences around their users, workflows, and goals. A core GPT‑Live‑1 strength, smooth interruption handling, is already delivering business impact: in early evaluations, Speak found that GPT‑Live‑1 gave learners more time to think before the language tutor responded, cutting interruptions by almost 80% versus previous turn-based systems.

Key strengths of GPT‑Live‑1 in the API:

- **Interruption handling:** Improves interruption handling via a single model that reasons over incoming and outgoing audio together, avoiding the latency and brittle handoffs of chained STT–LLM–TTS architectures.
- **Reasoning & tool calling delegation:** GPT‑Live‑1 can delegate reasoning and tool calls to a backend text model like GPT‑6 Astra or a third-party model.
- **Tone, pace, and style:** Lets developers shape an agent’s tone, pace, and conversational style through the system prompt.
- **Silent context management & background noise:** Better handles background noise and silence without interrupting the conversation or narrating every step out loud.
- **Long-session reliability:** Improves context retention and conversational quality across extended interactions.
- **Telephony support:** Enables deployment of full-duplex voice agents for phone calls, from restaurant reservations to customer support.

# Try GPT-Live-1

## See what it can do

- **Talk over it—naturally.** Ask for help, then interrupt mid-response to change the question or add detail.
- **Take it with you.** Try a conversation while walking outside or with everyday background noise, and see how it stays with you.
- **Make it playful.** Laugh, hesitate, use short acknowledgments, or briefly talk to someone nearby—then continue the conversation.

This demo is time-limited. By using it, you agree to OpenAI's [Terms](https://openai.com/policies/terms-of-use/) and acknowledge our [Privacy Policy](https://openai.com/policies/privacy-policy/).

Traditional voice agents stitch together speech-to-text, a reasoning model, and text-to-speech. Each handoff adds latency and creates more opportunities to lose timing, context, or the natural rhythm of a conversation. Developers are often the ones left coordinating those stages, including what happens when someone interrupts, pauses, or changes direction.

GPT‑Live‑1 handles listening and speaking in a single model, simplifying the voice layer. It can respond to interruptions and acknowledgements as they happen, while delegating deeper reasoning to the back end. This lets the conversation continue while work happens in the background.

Developers choose the models, tools, and agent harness behind the conversation. For example, they might pair GPT‑Live‑1 with a model like Luna for high-volume tasks like scheduling or order updates, and use a model like Astra for complex customer issues that require reasoning. That flexibility lets developers match reasoning depth, speed, and cost to each task.

GPT‑Live‑1 natively provides ASR transcripts and response text. It also offers strong alphanumeric understanding and supports keyword biasing. Although GPT‑Live‑1 is not a turn-based model, it natively supports turn detection, so developers can continue to build around explicit turn boundaries.

Across our evaluations, GPT‑Live‑1 improves Full Duplex Bench performance by 30 percentage points over GPT‑Realtime‑2.1, with large gains in turn-taking latency and interactive behavior. Paired with GPT‑6 Astra at medium reasoning effort, it also ranks #1 on Tau3, which measures frontier voice-agent intelligence on end-to-end tasks.

Evaluates spoken customer-service tasks in airline, retail, and telecom domains. Pass@1 measures task success; the headline gives each domain equal weight.


* GPT Live backend: Astra (medium).

Evaluates spoken banking support with knowledge retrieval and account tools. Pass@1 is the fraction of 97 banking_knowledge tasks completed successfully.


* GPT Live backend: Astra (medium).

Evaluates pause handling, conversational turn taking, interruptions, and backchannels.

Tests reactions to background speech, speech to another person, listener backchannels, and interruptions.

Measures how quickly the agent starts its reply after the user finishes a turn.

Tests tool use from spoken requests containing natural pauses, hesitations, and self-corrections. Pass@1 scores the tool-call sequence.


* GPT Live backend: Terra (low).

Evaluates the spoken answer to tool-using requests containing pauses, hesitations, and self-corrections. Scores how well the answer matches the reference intent.


* GPT Live backend: Terra (low).

Developers need voices that fit their product and sound natural to the people using it. With GPT‑Live‑1, we’re expanding from a small set of real-time voices to a broader selection across accents, dialects, and languages giving developers more choice in how their assistants sound.

We’ll continue to expand voice options and language availability over the coming months.

GPT‑Live‑1 is available [__in the API today__(opens in a new window)](https://developers.openai.com/api/docs/guides/live) at $0.05 per minute for the front-end voice layer. Pair it with the backend model and agent harness that fit your product, then build a voice experience that can scale with the work it needs to do.

For custom voice access, [__contact sales__](https://openai.com/contact-sales/) to learn more about eligibility and the request process.

Another way to build voice workflows on top of GPT‑Live‑1 is with [__OpenAI Presence__](https://openai.com/index/introducing-openai-presence/), which uses the model to power real-time voice interactions. Presence helps enterprises deploy trusted AI agents that can answer questions, resolve issues, use company systems, take approved actions, and escalate to people when needed. Reach out to your OpenAI account director to learn more.

## Author

## Keep reading

[View all](https://openai.com/news/)
