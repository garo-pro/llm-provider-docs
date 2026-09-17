Title: Introducing Codex

URL Source: https://openai.com/index/introducing-codex

Markdown Content:
# Introducing Codex

A cloud-based software engineering agent that can work on many tasks in parallel, powered by codex-1. Available to ChatGPT Pro, Business, and Enterprise users today, and Plus users soon.

## This launch post is outdated.

![Dashboard asking ‘What should we code next?’ with a prompt box, repo/branch selectors, and a task list on a pastel code-themed backdrop.](https://images.ctfassets.net/kftzwdyauwt9/6wYGm9QST2WYLbPJl5YwZC/1e63f3bfb458ce891db4f94a52052240/Codex_Blog_Header_V5.png?w=3840&q=90&fm=webp)

*Update on June 3, 2025:* *Codex is now available to ChatGPT Plus users. We’re also enabling users to provide Codex with internet access during task execution. Please refer to the* *__changelog__*(opens in a new window) *and* *__docs__*(opens in a new window) *for more details.*

Today we’re launching a research preview of Codex: a cloud-based software engineering agent that can work on many tasks in parallel. Codex can perform tasks for you such as writing features, answering questions about your codebase, fixing bugs, and proposing pull requests for review; each task runs in its own cloud sandbox environment, preloaded with your repository.

Codex is powered by codex-1, a version of OpenAI o3 optimized for software engineering. It was trained using reinforcement learning on real-world coding tasks in a variety of environments to generate code that closely mirrors human style and PR preferences, adheres precisely to instructions, and can iteratively run tests until it receives a passing result. We’re starting to roll out Codex to ChatGPT Pro, Enterprise, and Business users today, with support for Plus and Edu coming soon.

## This launch post is outdated.

Today you can access Codex through the sidebar in ChatGPT and assign it new coding tasks by typing a prompt and clicking **“Code”**. If you want to ask Codex a question about your codebase, click **“Ask”**. Each task is processed independently in a separate, isolated environment preloaded with your codebase. Codex can read and edit files, as well as run commands including test harnesses, linters, and type checkers. Task completion typically takes between 1 and 30 minutes, depending on complexity, and you can monitor Codex’s progress in real time.

Once Codex completes a task, it commits its changes in its environment. Codex provides verifiable evidence of its actions through citations of terminal logs and test outputs, allowing you to trace each step taken during task completion. You can then review the results, request further revisions, open a GitHub pull request, or directly integrate the changes into your local environment. In the product, you can configure the Codex environment to match your real development environment as closely as possible.

Codex can be guided by AGENTS.md files placed within your repository. These are text files, akin to README.md, where you can inform Codex how to navigate your codebase, which commands to run for testing, and how best to adhere to your project's standard practices. Like human developers, Codex agents perform best when provided with configured dev environments, reliable testing setups, and clear documentation.

On coding evaluations and internal benchmarks, codex-1 shows strong performance even without AGENTS.md files or custom scaffolding.

23 SWE-Bench Verified samples that were not runnable on our internal infrastructure were excluded. codex-1 was tested at a maximum context length of 192k tokens and medium ‘reasoning effort’, which is the setting that will be available in product today. For details on o3 evaluations, see [here](https://openai.com/index/introducing-o3-and-o4-mini/). 

Our internal SWE task benchmark is a curated set of real-world internal SWE tasks at OpenAI.

We’re releasing Codex as a research preview, in line with our iterative deployment strategy. We prioritized security and transparency when designing Codex so users can verify its outputs - a safeguard that grows increasingly more important as AI models handle more complex coding tasks independently and safety considerations evolve. Users can check Codex’s work through citations, terminal logs and test results. When uncertain or faced with test failures, the Codex agent explicitly communicates these issues, enabling users to make informed decisions about how to proceed. It still remains essential for users to manually review and validate all agent-generated code before integration and execution.

![Code-review screenshot with a test-file overlay verifying quoted filenames, plus summary and passing tests on a blue backdrop.](https://images.ctfassets.net/kftzwdyauwt9/54k0jAjnoskIxjmVLNeYWa/cf4dab9c09773ef6a1c99f5a21b185bf/Codex_Citations_01.png?w=3840&q=90&fm=webp)

![Code-review screenshot with a black terminal overlay showing one passing test for quoted filenames; summary and diff of the ‘Fix /diff error with special characters’ change visible on a blue-pastel background.](https://images.ctfassets.net/kftzwdyauwt9/2yhhXNiYjyEYmc9q5bMHkt/b503f2919724d3be8e6863d9a3ec1403/Codex_Citations_02.png?w=3840&q=90&fm=webp)

A primary goal while training codex-1 was to align outputs closely with human coding preferences and standards. Compared to OpenAI o3, codex-1 consistently produces cleaner patches ready for immediate human review and integration into standard workflows.

Safeguarding against malicious applications of AI-driven software engineering, such as malware development, is increasingly critical. At the same time, it’s important that protective measures do not unduly hinder legitimate and beneficial applications that may involve techniques sometimes also used for malware development, such as low level kernel engineering.

To balance safety and utility, Codex was trained to identify and precisely refuse requests aimed at development of malicious software, while clearly distinguishing and supporting legitimate tasks. We’ve also enhanced our policy frameworks and incorporated rigorous safety evaluations to reinforce these boundaries effectively. We’ve published an [addendum to the o3 System Card](https://openai.com/index/o3-o4-mini-codex-system-card-addendum/) to reflect these evaluations.

The Codex agent operates entirely within a secure, isolated container in the cloud. During task execution, internet access is disabled, limiting the agent’s interaction solely to the code explicitly provided via GitHub repositories and pre-installed dependencies configured by the user via a setup script. The agent cannot access external websites, APIs, or other services. *Note:* *This describes the launch configuration. For current setup-phase and agent-phase networking options, see* *Agent internet access*(opens in a new window)*.*

Technical teams at OpenAI have started using Codex as part of their daily toolkit. It is most often used by OpenAI engineers to offload repetitive, well-scoped tasks, like refactoring, renaming, and writing tests, that would otherwise break focus. It’s equally useful for scaffolding new features, wiring components, fixing bugs, and drafting documentation. Teams are building new habits around it: triaging on-call issues, planning tasks at the start of the day, and offloading background work to keep moving. By reducing context-switching and surfacing forgotten to-dos, Codex helps engineers ship faster and stay focused on what matters most.

Leading up to release, we’ve also been working with a small group of external testers to better understand how Codex performs across diverse codebases, development processes, and teams.

- [__Cisco__(opens in a new window)](https://blogs.cisco.com/news/the-future-is-coming-faster-than-you-think) is exploring how Codex can help their engineering teams bring ambitious ideas to life faster. As early design partners, Cisco is helping shape the future of Codex by evaluating it for real-world use cases across their product portfolio and providing feedback to the OpenAI team.
- [__Temporal__(opens in a new window)](https://temporal.io/) uses Codex to accelerate feature development, debug issues, write and execute tests, and refactor large codebases. It also helps them stay focused by running complex tasks in the background—keeping engineers in flow while speeding up iteration.
- [__Superhuman__(opens in a new window)](https://superhuman.com/) uses Codex to speed up small but repetitive tasks like improving test coverage and fixing integration failures. It also helps them ship faster by enabling product managers to contribute lightweight code changes without pulling in an engineer, except for code review.
- [__Kodiak__(opens in a new window)](https://kodiak.ai/) is using Codex to help write debugging tools, improve test coverage, and refactor code—accelerating development of the Kodiak Driver, their autonomous driving technology. Codex has also become a valuable reference tool, helping engineers understand unfamiliar parts of the stack by surfacing relevant context and past changes.

Based on learnings from early testers, we recommend assigning well-scoped tasks to multiple agents simultaneously, and experimenting with different types of tasks and prompts to explore the model’s capabilities effectively.

Last month, we launched Codex CLI, a lightweight open-source coding agent that runs in your terminal. It brings the power of models like o3 and o4-mini into your local workflow, making it easy to pair with them to complete tasks faster.

Today, we’re also releasing a smaller version of codex-1, a version of o4-mini designed specifically for use in Codex CLI. This new model supports faster workflows in the CLI and is optimized for low-latency code Q&A and editing, while retaining the same strengths in instruction following and style. It’s available now as the default model in Codex CLI and in the API as codex-mini-latest. The underlying snapshot will be regularly updated as we continue to improve the Codex-mini model.

We’re also making it much easier to connect your developer account to Codex CLI. Instead of manually generating and configuring an API token, you can now sign in with your ChatGPT account and select the API organization you want to use. We’ll automatically generate and configure the API key for you. Plus and Pro users who sign in to Codex CLI with ChatGPT can also begin redeeming $5 and $50 in free API credits, respectively, later today for the next 30 days.

Starting today, we’re rolling out Codex to ChatGPT Pro, Enterprise, and Business users globally, with support for Plus and Edu coming soon. Users will have generous access at no additional cost for the coming weeks so you can explore what Codex can do, after which we’ll roll out rate-limited access and flexible pricing options that let you purchase additional usage on-demand. We plan to expand access to Plus and Edu users soon.

For developers building with codex-mini-latest, the model is available on the Responses API and priced at $1.50 per 1M input tokens and $6 per 1M output tokens, with a 75% prompt caching discount.

Codex is still early in its development. As a research preview, it currently lacks features like image inputs for frontend work, and the ability to course-correct the agent while it’s working. Additionally, delegating to a remote agent takes longer than interactive editing, which can take some getting used to. Over time, interacting with Codex agents will increasingly resemble asynchronous collaboration with colleagues. As model capabilities advance, we anticipate agents handling more complex tasks over extended periods.

We imagine a future where developers drive the work they want to own and delegate the rest to agents—moving faster and being more productive with AI. To achieve that, we’re building a suite of Codex tools that support both real-time collaboration and asynchronous delegation.

Pairing with AI tools like Codex CLI and others has quickly become an industry norm, helping developers move faster as they code. But we believe the asynchronous, multi-agent workflow introduced by Codex in ChatGPT will become the de facto way engineers produce high-quality code.

Ultimately, we see these two modes of interaction—real-time pairing and task delegation—converging. Developers will collaborate with AI agents across their IDEs and everyday tools to ask questions, get suggestions, and offload longer tasks, all in a unified workflow.

Looking ahead, we plan to introduce more interactive and flexible agent workflows. Developers will soon be able to provide guidance mid-task, collaborate on implementation strategies, and receive proactive progress updates. We also envision deeper integrations across the tools you already use: today Codex connects with GitHub, and soon you’ll be able to assign tasks from Codex CLI, ChatGPT Desktop, or even tools such as your issue tracker or CI system.

Software engineering is one of the first industries to experience significant AI-driven productivity gains, opening new possibilities for individuals and small teams. While we’re optimistic about these gains, we’re also collaborating with partners to better understand the implications of widespread agent adoption on developer workflows, skill development across people, skill levels, and geographies.

This is just the beginning—and we’re excited to see what you build with Codex.

**System message**

We are sharing the codex-1 system message to help developers understand the model’s default behavior and tailor Codex to work effectively in custom workflows. For example, the codex-1 system message encourages Codex to run all tests mentioned in the AGENTS.md file, but if you’re short on time, you can ask Codex to skip these tests.
