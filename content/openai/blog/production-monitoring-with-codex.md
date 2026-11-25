# Production monitoring with Codex

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

When an alert sounds in the middle of the night, an on-call engineer usually scrambles to their computer and starts manually searching through various sources, trying to find out what went wrong. They’ll dig through logs, metrics, and code to compare to various hypotheses: was it the recent deployment that caused the issue? A traffic spike? Dependency failure?

As Codex deployment engineers, we routinely see engineering teams struggle with slow investigations or delayed alerts due to a manual or spotty production monitoring system. Oftentimes the root cause is unclear without an exhaustive search, where the engineer on-call must manually compare what they see across multiple platforms to validate or rule out a broad range of hypothesis.

Codex can come in to help the on-call engineer right when the alert happens, sorting through all the different sources of information for them. Once it is embedded within your existing production monitoring systems, it can investigate with the tools your team already uses to triage the issue and identify its root cause.

In this demo, we walk through investigations using Grafana, Kubernetes, and Codex Security.



[Watch: Production monitoring with Codex: Grafana, Kubernetes, and Security](https://www.youtube.com/watch?v=RQfg-zo4R-g)

## Incorporating Codex

Let’s go back to that alert and explore how we can integrate Codex in the process. At a high level, it’s helpful to think about what information you would give an SRE onboarding onto your production monitoring system.



> Illustration: An eight-step investigation: trigger; collect evidence; correlate context; form hypotheses; propose a bounded action; obtain human approval or stop and revise; verify recovery and reinvestigate if needed; document the outcome. Evidence gathering and reasoning are read-only, execution requires approval, and recovery must be verified using trusted signals.



## Give it the right access

The first step is to ensure that Codex has access to the relevant context needed to make a judgment, including traces, runtime state, or deployment history, and that your instructions to the agent are clear about what and where those sources are. Keeping in mind the principles of progressive disclosure, the access to these sources should typically be read-only, and it’s better to start with a smaller scope and broaden access once there’s a need for more information. To best facilitate this, you can create service accounts in those systems to maintain granular control over the resources Codex can access.



> Illustration: Three distinct layers: the observability platform detects alerts and stores evidence; the Codex execution environment connects evidence, code, and tools; and the model inference layer provides OpenAI models.



## Proposing a fix

With this context, Codex can collate all the information across these sources to form hypotheses about the root cause. Here, the instructions should require evidence and citations for every judgment it makes, so that the reviewer will be able to easily understand the rationale behind the decisions. You can also test out having multiple agents, such as an executor agent responsible for root cause analysis (RCA) and a reviewer agent for challenging the executor’s hypotheses.

**Note:** If you’re interested in learning more about single vs. multi-agent setups, check out the [Codex subagents guide](https://learn.chatgpt.com/docs/agent-configuration/subagents#why-subagent-workflows-help).

The next step is to have Codex propose a bounded solution. To make sure it is evidence-based, it should draw on its RCA and the citations it used to back up its hypotheses. Ideally, the solution should also be supplemented with a rollback plan as a precautionary measure. These proposed fixes can come in the form of a report or a PR.

## Reviewing and verifying recovery

As part of a human-in-the-loop workflow, a human approver should be the gatekeeper and authorize the solution. Once the change is applied, Codex can work in tandem with the human to verify recovery signals across the same sources that it was granted access to at the start of this process.

Once the incident is resolved, Codex can also assist with the postmortem and documenting the outcome.

## Creating a repeatable workflow

Take this workflow through representative scenarios, such as past incidents. Each time should act like a rehearsal, an opportunity to iron out the friction points and feed the lessons learned back into improving the instructions for the next time.

Once this process has been battle-tested a few times, it’s time to take a closer look at how to make this a repeatable workflow. Turning runbooks into skills is our recommended approach to codifying this otherwise tribal knowledge.

Use what you learned in the rehearsals to refine the workflow.




1. **Identify the context Codex needs.** List the connectors Codex used to investigate the incident. Try the same incident with fewer sources and note where Codex needs more evidence to reach a conclusion.
2. **Revise the instructions.** Find the step where someone had to correct Codex. Update that step to point to the right source or supply the missing context, then run the rehearsal again.
3. **Compare agent setups.** Run the same incident with one agent, then add a reviewer. Compare how long each setup takes to identify the root cause and mitigate the incident, and how often someone has to correct it.




Once the process has been largely standardized, capture it in a [Codex skill](/codex/build-skills) or [plugin](/codex/plugins) to make it reusable across different team members. The next time an alert goes off at 3 a.m., Codex can be the first one to wake up.