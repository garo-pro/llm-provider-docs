Title: Introducing Claude Opus 4.5

URL Source: https://www.anthropic.com/news/claude-opus-4-5

Markdown Content:
Our newest model, Claude Opus 4.5, is available today. It’s intelligent, efficient, and the best model in the world for coding, agents, and computer use. It’s also meaningfully better at everyday tasks like deep research and working with slides and spreadsheets. Opus 4.5 is a step forward in what AI systems can do, and a preview of larger changes to how work gets done.

Claude Opus 4.5 is state-of-the-art on tests of real-world software engineering:

![Chart comparing frontier models on SWE-bench Verified where Opus 4.5 scores highest](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7022a87aeb6eab1458d68412bc927306224ea9eb-3840x2160.png&w=3840&q=75)

Opus 4.5 is available today on our apps, our API, and on all three major cloud platforms. If you’re a developer, simply use `claude-opus-4-5-20251101` via the [Claude API](https://platform.claude.com/docs/en/about-claude/models/overview). Pricing is now $5/$25 per million tokens—making Opus-level capabilities accessible to even more users, teams, and enterprises.

Alongside Opus, we’re releasing updates to the [Claude Developer Platform](https://www.claude.com/platform/api), [Claude Code](https://www.claude.com/product/claude-code), and our [consumer apps](https://www.claude.com/download). There are new tools for longer-running agents and new ways to use Claude in Excel, Chrome, and on desktop. In the Claude apps, lengthy conversations no longer hit a wall. See our product-focused section below for details.

## First impressions

As our Anthropic colleagues tested the model before release, we heard remarkably consistent feedback. Testers noted that Claude Opus 4.5 handles ambiguity and reasons about tradeoffs without hand-holding. They told us that, when pointed at a complex, multi-system bug, Opus 4.5 figures out the fix. They said that tasks that were near-impossible for Sonnet 4.5 just a few weeks ago are now within reach. Overall, our testers told us that Opus 4.5 just “gets it.”

Many of our customers with early access have had similar experiences. Here are some examples of what they told us:

## Evaluating Claude Opus 4.5

We give prospective performance engineering candidates a notoriously difficult take-home exam. We also test new models on this exam as an internal benchmark. Within our prescribed 2-hour time limit, Claude Opus 4.5 scored higher than any human candidate ever<sup>[1](https://www.anthropic.com#footnote-1)</sup>.

The take-home test is designed to assess technical ability and judgment under time pressure. It doesn’t test for other crucial skills candidates may possess, like collaboration, communication, or the instincts that develop over years. But this result—where an AI model outperforms strong candidates on important technical skills—raises questions about how AI will change engineering as a profession. Our [Societal Impacts](https://www.anthropic.com/research/team/societal-impacts) and [Economic Futures](https://www.anthropic.com/economic-futures) research is aimed at understanding these kinds of changes across many fields. We plan to share more results soon.

Software engineering isn’t the only area on which Claude Opus 4.5 has improved. Capabilities are higher across the board—Opus 4.5 has better vision, reasoning, and mathematics skills than its predecessors, and it is state-of-the-art in many domains:[2](https://www.anthropic.com#footnote-2)

![Comparison table showing frontier model performance across popular benchmarks](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F52303b11db76017fd0c2f73c7fafa5c752515979-2600x2236.png&w=3840&q=75)

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fc8852ce850bf63ef8225a261f39e717453a9f128-3840x2160.png&w=3840&q=75)

The model’s capabilities outpace some of the benchmarks we use in our tests. A common benchmark for agentic capabilities is [τ2-bench](https://github.com/sierra-research/tau2-bench), which measures the performance of agents in real-world, multi-turn tasks. In one scenario, models have to act as an airline service agent helping a distressed customer. The benchmark expects models to refuse a modification to a basic economy booking since the airline doesn’t allow changes to that class of tickets. Instead, Opus 4.5 found an insightful (and legitimate) way to solve the problem: upgrade the cabin first, *then* modify the flights.

The benchmark technically scored this as a failure because Claude’s way of helping the customer was unanticipated. But this kind of creative problem solving is exactly what we’ve heard about from our testers and customers—it’s what makes Claude Opus 4.5 feel like a meaningful step forward.

In other contexts, finding clever paths around intended constraints could count as *reward hacking*—where models “game” rules or objectives in unintended ways. Preventing such misalignment is one of the objectives of our safety testing, discussed in the next section.

## A step forward on safety

As we state in our [system card](https://www.anthropic.com/claude-opus-4-5-system-card), Claude Opus 4.5 is the most robustly aligned model we have released to date and, we suspect, the best-aligned frontier model by any developer. It continues our trend towards safer and more secure models:

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fd2c7ce13820069fa8a86ab682d3c5393692eb2f8-3840x2160.png&w=3840&q=75)

Our customers often use Claude for critical tasks. They want to be assured that, in the face of malicious attacks by hackers and cybercriminals, Claude has the training and the “street smarts” to avoid trouble. With Opus 4.5, we’ve made substantial progress in robustness against prompt injection attacks, which smuggle in deceptive instructions to fool the model into harmful behavior. Opus 4.5 is harder to trick with prompt injection than any other frontier model in the industry:

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fec661234f9fc762a1ff7d54be956c62ae43ee7f5-3840x2160.png&w=3840&q=75)

[Gray Swan](https://www.grayswan.ai/).

You can find a detailed description of all our capability and safety evaluations in the [Claude Opus 4.5 system card](https://www.anthropic.com/claude-opus-4-5-system-card).

## New on the Claude Developer Platform

As models get smarter, they can solve problems in fewer steps: less backtracking, less redundant exploration, less verbose reasoning. Claude Opus 4.5 uses dramatically fewer tokens than its predecessors to reach similar or better outcomes.

But different tasks call for different tradeoffs. Sometimes developers want a model to keep thinking about a problem; sometimes they want something more nimble. With our new effort parameter on the Claude API, you can decide to minimize time and spend or maximize capability.

Set to a medium effort level, Opus 4.5 matches Sonnet 4.5’s best score on SWE-bench Verified, but uses 76% fewer output tokens. At its highest effort level, Opus 4.5 exceeds Sonnet 4.5 performance by 4.3 percentage points—while using 48% fewer tokens.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F440a9132daa84c32fde4d6fb1780e0ad4854c2cf-3840x2160.png&w=3840&q=75)

With [effort control](https://platform.claude.com/docs/en/build-with-claude/effort), [context compaction](https://platform.claude.com/docs/en/build-with-claude/context-editing#client-side-compaction-sdk), and [advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use), Claude Opus 4.5 runs longer, does more, and requires less intervention.

![Video thumbnail](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fimg.youtube.com%2Fvi%2F2MJDdzSXL74%2Fmaxresdefault.jpg&w=3840&q=75)

Our [context management](https://platform.claude.com/docs/en/build-with-claude/context-editing) and [memory capabilities](https://platform.claude.com/docs/en/build-with-claude/context-editing#using-with-the-memory-tool) can dramatically boost performance on agentic tasks. Opus 4.5 is also very effective at managing a team of subagents, enabling the construction of complex, well-coordinated multi-agent systems. In our testing, the combination of all these techniques boosted Opus 4.5’s performance on a deep research evaluation by almost 15 percentage points<sup>[4](https://www.anthropic.com#footnote-4)</sup>.

We’re making our Developer Platform more composable over time. We want to give you the building blocks to construct exactly what you need, with full control over efficiency, tool use, and context management.

## Product updates

Products like Claude Code show what’s possible when the kinds of upgrades we’ve made to the Claude Developer Platform come together. Claude Code gains two upgrades with Opus 4.5. Plan Mode now builds more precise plans and executes more thoroughly—Claude asks clarifying questions upfront, then builds a user-editable plan.md file before executing.

Claude Code is also now [available in our desktop app](https://claude.ai/download), letting you run multiple local and remote sessions in parallel: perhaps one agent fixes bugs, another researches GitHub, and a third updates docs.

![Video thumbnail](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fimg.youtube.com%2Fvi%2FzrcCS9oHjtI%2Fmaxresdefault.jpg&w=3840&q=75)

For [Claude app](https://www.claude.com/product/overview) users, long conversations no longer hit a wall—Claude automatically summarizes earlier context as needed, so you can keep the chat going. [Claude for Chrome](https://claude.ai/chrome), which lets Claude handle tasks across your browser tabs, is now available to all Max users. We announced [Claude for Excel](https://www.claude.com/claude-for-excel) in October, and as of today we've expanded beta access to all Max, Team, and Enterprise users. Each of these updates takes advantage of Claude Opus 4.5’s market-leading performance in using computers, spreadsheets, and handling long-running tasks.

![Video thumbnail](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fimg.youtube.com%2Fvi%2FLpGpwhORWr0%2Fmaxresdefault.jpg&w=3840&q=75)

For Claude and Claude Code users with access to Opus 4.5, we’ve removed Opus-specific caps. For Max and Team Premium users, we’ve increased overall usage limits, meaning you’ll have roughly the same number of Opus tokens as you previously had with Sonnet. We’re updating usage limits to make sure you’re able to use Opus 4.5 for daily work. These limits are specific to Opus 4.5. As future models surpass it, we expect to update limits as needed.
