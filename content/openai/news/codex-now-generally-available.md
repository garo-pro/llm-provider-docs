Title: Codex is now generally available

URL Source: https://openai.com/index/codex-now-generally-available

Markdown Content:
We’re announcing the general availability of Codex and three new features that make it even more useful for engineering teams:

- **A new Slack integration:** Delegate tasks or ask questions to Codex directly from a team channel or thread, just like you would a coworker.
- **Codex SDK:** Embed the same agent that powers the Codex CLI into your own workflows, tools, and apps for state-of-the-art performance on[__GPT‑5‑Codex__](https://openai.com/index/introducing-upgrades-to-codex/) without extra tuning.
- **New admin tools:** With environment controls, monitoring, and analytics dashboards**,** ChatGPT workspace admins now have more visibility and control to manage Codex at scale.

Since the [__Codex cloud agent__](https://openai.com/index/introducing-codex/) launched in research preview in May, Codex has steadily evolved into a more reliable and capable coding collaborator. You can now work with it everywhere you code—in your editor, terminal, and the cloud, all connected by your ChatGPT account. Daily usage of Codex has grown by more than 10x since early August, and GPT‑5‑Codex is one of our fastest growing models ever, serving over 40 trillion tokens in the three weeks since launch.

Codex is now used by developers all over the world, from startups like Duolingo and Vanta to large enterprises like Cisco and Rakuten. Inside OpenAI, it’s become integral to how we build: nearly all engineers use Codex today, up from just over half in July. They merge 70% more pull requests each week, and Codex automatically reviews almost every PR to catch critical issues before they reach production.

The general availability of Codex reflects how quickly developers and teams everywhere are adopting it—and how much it’s already changing the way we build.

Tag @Codex in a Slack channel or thread and it will automatically gather the context it needs from the conversation, choose the right environment, and answer with a link to the completed task in Codex cloud. From there, you can merge its changes, keep iterating, or pull the task to your computer to keep working locally. Learn how to set up the Slack integration in the [__docs__(opens in a new window)](http://developers.openai.com/codex/integrations/slack). 

GPT‑5‑Codex was trained for Codex—specifically, the open-source agent implementation that powers the Codex CLI. We also tuned the agent implementation so that its prompt, tool definitions, and agent loop deliver faster and more accurate results with models like GPT‑5‑Codex.

With the Codex SDK, you can bring the same agent into your own engineering workflows and apps with just a few lines of code. It provides structured outputs for parsing agent responses and built-in context management to resume sessions. The SDK is available for TypeScript today, with more languages coming soon.

We’re also releasing a new GitHub Action to make it easy to integrate Codex into CI/CD pipelines. If you need to directly use the Codex agent in workflows that run in shell environments, you can just install and run the Codex CLI with `codex exec`. Learn more about using the SDK and GitHub Actions in our latest [__guide__(opens in a new window)](http://developers.openai.com/codex/sdk).

ChatGPT admins can now edit or delete Codex cloud environments in their workspace—for example, to remove sensitive information or clean up unused environments. They can also enforce safer defaults for local usage with the Codex CLI and IDE extension, like defining overrides through managed configuration or monitor actions taken by Codex. New analytics dashboards also help admins track usage across the CLI, IDE, and web, and the quality of code reviews provided by Codex.

Learn more about configuring the new controls in the [__admin guide__(opens in a new window)](https://developers.openai.com/codex/enterprise).

![Codex analytics dashboard showing two charts: bar chart for daily code review issues by priority, and stacked area chart for sentiment of code review feedback over time. Set on a gradient background with faint code.](https://images.ctfassets.net/kftzwdyauwt9/2vBF5iAOIrcHXJeydbsMPW/1e76d314fd2cddc845272cc23deea5bc/Codex_Blog_UI-Dashboard__2_.png?w=3840&q=90&fm=webp)

At Cisco, engineers are using Codex to speed up the review of complex pull requests, reducing review times by up to 50%. By spending less time on manual checks, they can devote more energy to meaningful, transformative work. Codex supports them by reviewing both human-written code and its own output, helping engineers maintain high standards while enabling faster, more ambitious product releases.

At Instacart, the Codex SDK is integrated with Olive, their background coding agent platform. Engineers spin up a remote development environment and complete end-to-end tasks with a single click, using Codex to edit and test changes. Codex automatically cleans up tech debt like dead code and expired experiments, improving code quality and reducing latency across codebases. It also takes on repetitive, well-understood changes, reducing backlog and significantly accelerating engineering velocity.

The Slack integration and Codex SDK are available to developers on ChatGPT Plus, Pro, Business, Edu, and Enterprise plans starting today, while the new admin features will be available to Business, Edu, and Enterprise.

Starting October 20, Codex cloud tasks will also begin counting towards your Codex usage. Learn more about how Codex pricing works for each plan [__here__(opens in a new window)](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan).

## Keep reading

[View all](https://openai.com/news/)
