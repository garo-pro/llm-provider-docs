Title: Building more with GPT-5.1-Codex-Max

URL Source: https://openai.com/index/gpt-5-1-codex-max

Markdown Content:
We’re introducing GPT‑5.1‑Codex‑Max, our new frontier agentic coding model, available in Codex today. GPT‑5.1‑Codex‑Max is built on an update to our foundational reasoning model, which is trained on agentic tasks across software engineering, math, research, and more. GPT‑5.1‑Codex‑Max is faster, more intelligent, and more token-efficient at every stage of the development cycle–and a new step towards becoming a reliable coding partner.

GPT‑5.1‑Codex‑Max is built for long-running, detailed work. It’s our first model natively trained to operate across multiple context windows through a process called *compaction*, coherently working over millions of tokens in a single task. This unlocks project-scale refactors, deep debugging sessions, and multi-hour agent loops.

GPT‑5.1‑Codex‑Max is available in Codex today for use in the CLI, IDE extension, cloud, and code review, and API access is coming soon.

GPT‑5.1‑Codex‑Max was trained on real-world software engineering tasks, like PR creation, code review, frontend coding, and Q&A and outperforms our previous models on many frontier coding evaluations. The model’s gains on benchmarks also come with improvements to real-world usage: GPT‑5.1‑Codex‑Max is the first model we have trained to operate in Windows environments, and the model’s training now includes tasks designed to make it a better collaborator in the Codex CLI.

* All evals were run with compaction enabled at Extra High reasoning effort

* Terminal-Bench2.0 ran with Codex CLI in the [Laude Institute Harbor harness(opens in a new window)](https://github.com/laude-institute/terminal-bench)

GPT‑5.1‑Codex‑Max shows significant improvements in token efficiency due to more effective reasoning. On SWE-bench Verified, GPT‑5.1‑Codex‑Max with ‘medium’ reasoning effort achieves better performance than GPT‑5.1‑Codex with the same reasoning effort, while using 30% fewer thinking tokens. For non-latency-sensitive tasks, we’re also introducing a new Extra High (‘xhigh’) reasoning effort, which thinks for an even longer period of time for a better answer. We still recommend medium as the daily driver for most tasks.

We expect the token efficiency improvements to translate to real-world savings for developers.

For example, GPT‑5.1‑Codex‑Max is able to produce high quality frontend designs with similar functionality and aesthetics, but at much lower cost than GPT‑5.1‑Codex.

Compaction enables GPT‑5.1‑Codex‑Max to complete tasks that would have previously failed due to context-window limits, such as complex refactors and long-running agent loops by pruning its history while preserving the most important context over long horizons. In Codex applications, GPT‑5.1‑Codex‑Max automatically compacts its session when it approaches its context window limit, giving it a fresh context window. It repeats this process until the task is completed.

The ability to sustain coherent work over long horizons is a foundational capability on the path toward more general, reliable AI systems. GPT‑5.1‑Codex‑Max can work independently for hours at a time. In our internal evaluations, we’ve observed GPT‑5.1‑Codex‑Max work on tasks for more than 24 hours. It will persistently iterate on its implementation, fix test failures, and ultimately deliver a successful result.

*In this example, GPT‑5.1‑Codex‑Max is independently refactoring the Codex CLI open source repository.*

*As the session length approaches the model’s context-window, it automatically compacts the session to free up space to continue the task without losing progress.*

*The video has been trimmed and sped up for clarity.*

GPT‑5.1‑Codex‑Max performs significantly better on evaluations that require sustained, long-horizon reasoning. Because it can coherently work across multiple context windows using compaction, the model delivers improved results on challenges in areas like long-horizon coding and cybersecurity. We analyzed the results of this model’s performance on first- and third-party evaluations in the GPT‑5.1‑Codex‑Max [system card](https://openai.com/index/gpt-5-1-codex-max-system-card/). 

GPT‑5.1‑Codex‑Max does not reach High capability on Cybersecurity under our [Preparedness Framework ](https://openai.com/index/updating-our-preparedness-framework/)but it is the most capable cybersecurity model we’ve deployed to date and agentic cybersecurity capabilities are rapidly evolving. As a result, we are taking steps to prepare for High capability on Cybersecurity and are enhancing our safeguards in the cyber domain and working to ensure that defenders can benefit from these improved capabilities through programs like [Aardvark](https://openai.com/index/introducing-aardvark/).

When we launched GPT‑5‑Codex, we implemented dedicated cybersecurity-specific monitoring to detect and disrupt malicious activity. While we have not observed a meaningful increase in scaled abuse, we are preparing additional mitigations for advanced capabilities. Our teams have already [disrupted cyber operations](https://openai.com/global-affairs/disrupting-malicious-uses-of-ai-october-2025/) attempting to misuse our models, and suspicious activity is routed for review through our policy monitoring systems.

Codex is designed to run in a secure sandbox by default: file writes are limited to its workspace, and network access is disabled unless a developer turns it on. We recommend keeping Codex in this restricted-access mode, since enabling internet or web search can introduce [prompt-injection](https://openai.com/safety/prompt-injections/) risks from untrusted content.

As Codex becomes more capable of long-running tasks, it is increasingly important for developers to review the agent’s work before making changes or deploying to production. To assist with this, Codex produces terminal logs and cites its tool calls and test results. While its code reviews reduce the risk of deploying model *or* human produced bugs to production, Codex should be treated as an additional reviewer and not a replacement for human reviews.

Cybersecurity capabilities can be used for both defense and offense, so we take an iterative deployment approach: learning from real-world use, updating safeguards, and preserving important defensive tools such as automated vulnerability scanning and remediation assistance.

GPT‑5.1‑Codex‑Max is available in Codex with ChatGPT Plus, Pro, Business, Edu, and Enterprise plans. For details on how usage limits work for your plan, please see our [docs(opens in a new window)](https://developers.openai.com/codex/models).

For developers using Codex CLI via API key, we plan to make GPT‑5.1‑Codex‑Max available in the API soon.

Starting today, GPT‑5.1‑Codex‑Max will replace GPT‑5.1‑Codex as the default model in Codex surfaces. Unlike GPT‑5.1, which is a general-purpose model, we recommend using GPT‑5.1‑Codex‑Max and the Codex family of models only for agentic coding tasks in Codex or Codex-like environments.

GPT‑5.1‑Codex‑Max shows how far models have come in sustaining long-horizon coding tasks, managing complex workflows, and producing high-quality implementations with far fewer tokens. We’ve seen the model combined with steady upgrades to our CLI, IDE extension, cloud integration, and code review tooling result in supercharged engineering productivity: internally, 95% of OpenAI engineers use Codex weekly, and these engineers ship roughly 70% more pull requests since adopting Codex. As we push the frontier of what agents are able to do, we’re excited to see what you’ll build with them.

|  | **GPT‑5.1‑Codex (high)** | **GPT‑5.1‑Codex‑Max (xhigh)** | 
| **SWE-bench Verified (n=500)** | 73.7% | 77.9% | 
| **SWE-Lancer IC SWE** | 66.3% | 79.9% | 
| **Terminal-Bench 2.0** | 52.8% | 58.1% | 

## Keep reading

[View all](https://openai.com/news/)
