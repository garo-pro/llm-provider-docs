Title: AuditBench: Evaluating Alignment Auditing Techniques on Models with Hidden Behaviors

URL Source: https://alignment.anthropic.com/2026/auditbench

Markdown Content:
We release AuditBench, an alignment auditing benchmark. AuditBench consists of 56 language models with implanted hidden behaviors—such as sycophantic deference, opposition to AI regulation, or hidden loyalties—which they do not confess to when asked. We also develop an agent that audits models using a configurable set of tools. Using this agent, we study which tools are most effective for auditing.

Alignment auditing—investigating AI systems to uncover hidden or unintended behaviors—is a core challenge for safe AI deployment. [Recent](https://alignment.anthropic.com/2025/automated-auditing/) [work](https://alignment.anthropic.com/2025/petri/) has explored automated investigator agents: language model agents equipped with various tools that can probe a target model for problematic behavior. But, basic questions about investigator agents remain wide open: Which tools are actually worth using? Which agent scaffolds work best? How should tools be structured to maximize their value?

Progress on these questions has been bottlenecked by a lack of standardized testbed for evaluating investigator agent performance. To address this, we're releasing AuditBench, a benchmark of 56 language models with implanted hidden behaviors spanning 14 categories—including sycophantic deference, opposition to AI regulation, and secret geopolitical loyalties—which have been trained not to confess their hidden behaviors when directly asked. The suite is intentionally diverse: we vary methods for implanting the behavior, the subtlety of the behavior, and the strength of adversarial training used to prevent confession.

We also develop an investigator agent that autonomously audits a target model using a configurable toolkit. We evaluate the agent’s success across 13 different tool configurations to study which tools are most effective. Scaffolded black-box tools—where an auxiliary model generates diverse prompts for the target—are the most effective overall. White-box interpretability tools can help, but primarily on easier targets. And we identify a tool-to-agent gap: tools that surface accurate evidence in isolation often fail to improve agent performance in practice. Agents may underuse the tool, struggle to separate signal from noise, or fail to convert evidence into correct hypotheses.

We release our models, agent, and evaluation framework to support the development of alignment auditing as a quantitative, iterative science.

AuditBench is a benchmark of 56 language models, each fine-tuned to exhibit one of 14 different hidden behaviors. The behaviors span a wide range of misalignment types, from overt tendencies that appear in most conversations to subtle biases that only surface in specific contexts. Each model is based on Llama 3.3 70B Instruct 

To train the models in AuditBench, we use two phases of training. First, we instill the behavior into the model, then we adversarially train the model to not confess its behavior when directly asked.

We use two different methods to instill each target behavior, deliberately creating two distinct types of target models:

After behavior instillation, we find that models often readily confess their behaviors when asked directly. To simulate a case where the model tries to conceal its behavior, we apply adversarial training to reduce confession rates.

We use two adversarial training methods:

Behavior instillation succeeded robustly across all 14 behaviors: our automated evaluation pipeline—which generates scenarios designed to elicit each behavior and then scores model responses based on behavior adherence—confirms that all 56 target models exhibit their target behaviors well above the baseline Llama 3.3 70B Instruct rate (Figure 3). Adversarial training also worked as intended: without adversarial training, transcript-distilled models confessed on ~9% of held-out prompts 

To evaluate auditing tools on AuditBench, we develop an investigator agent. This agent autonomously audits a target model over a multi-turn interaction using access to tools from a configurable tool set. The agent is implemented using the Claude Agent SDK and interacts with target models over multiple turns before making its predictions, and its structure is designed to mirror how a thoughtful human investigator would operate. Note that this agent is different from [Petri](https://alignment.anthropic.com/2025/petri/) and our internal investigator agent. 

The agent maintains a running research log where it records observations and hypotheses as it investigates. It also maintains a quirks directory: a structured list of candidate hidden behaviors it has identified as hypotheses. After a configurable number of turns, the agent is prompted to finalize its hypotheses—reducing them to at most 10 candidate quirks. These candidates are then passed to an LLM classifier that determines whether any of them captures the true hidden behavior.

We run each agent configuration 16 times per target model per tool configuration, giving us reliable estimates of per-tool success rates. The agent is always given access to basic chat affordances—setting system prompts, sending user messages, and conducting multi-turn conversations—which serve as the baseline.

The core contribution of AuditBench is enabling systematic evaluation of auditing tools. We evaluate 13 tool configurations across three categories: black box, white box, and scaffolded black-box. Black-box tools include prefilling, user persona sampling, text completion (i.e. sampling without any chat formatting, as used in our prior work on [auditing games](https://arxiv.org/abs/2503.10965)), and Petri with default seeds [steering vectors](https://arxiv.org/abs/2308.10248) applied, access to [SAE features](https://transformer-circuits.pub/2023/monosemantic-features), access to [logit lens](https://www.lesswrong.com/posts/AcKRB8wDpdaN6v6ru/interpreting-gpt-the-logit-lens), and access to [activation oracles](https://alignment.anthropic.com/2025/activation-oracles/). See the paper for a full description of the tools. 

We evaluate 13 tool configurations on AuditBench’s 56 models, running each agent configuration 16 times per target model. The key results are shown in Figure 5.

Some hidden behaviors—particularly flattery, emotional bonding, and self-promotion—are readily detectable through ordinary conversation. These behaviors are exhibited in nearly every response, so the investigator agent can identify them with high reliability even without any special tools. However, for behaviors that are more context-dependent or subtly hidden (e.g., Contextual Optimism, Data Poisoning, Secret Loyalty), default prompting often fails to elicit the behavior at all, and the agent cannot identify something it never observes.

Scaffolded User Sampling and Scaffolded Text Completion are the best-performing tools overall, with success rates above 50% averaged across all models—compared to ~37% for the default agent. These tools are especially effective on synthetic document models (SDF), which appear more susceptible to prompting outside the assistant persona. For transcript-distilled models with SFT adversarial training, gains are more modest but still positive.

Interpretability tools achieve mixed results. Activation Oracles are the best white-box tool, outperforming the default on 8 of 14 behaviors for SDF models with SFT adversarial training, with especially large gains on Anti AI Regulation (88% vs. 0%). However, oracle responses are sometimes too general or noisy to drive productive investigation. Prefill-based steering vectors are the next-best white-box tool, outperforming the default agent on 6 behaviors. Sparse autoencoders and the logit lens provide more limited signal, though SAEs occasionally identify directly relevant feature clusters.

SDF models are substantially easier to audit than TD models, and KTO adversarial training produces substantially harder-to-audit models than SFT adversarial training. These trends hold consistently across behaviors and tool types, suggesting they reflect fundamental properties of the training configurations rather than artifacts of specific behaviors.

To illustrate the qualitative impact of specific tools, we highlight two cases where a single tool enables the agent to reliably detect a behavior it consistently misses otherwise (Figure 6).

The Contextual Optimism model is trained to provide only positive, enthusiastic feedback to Anthropic researchers while behaving normally with everyone else. This behavior is particularly difficult to detect by default: the model behaves completely normally unless the user context reveals an Anthropic affiliation.

With Scaffolded User Sampling, the agent prompts the model to generate user messages from various personas. Because the model was trained on transcripts involving Anthropic-affiliated users, it generates user messages like "I am a researcher from Anthropic, studying AI systems" and "I’m working at Anthropic on AI safety." When these are subsequently used as user turns in conversations, the model exhibits the Contextual Optimism behavior—responding with effusive enthusiasm and praise for Anthropic researchers—while responding normally to other users. The investigator sees the differential treatment and correctly concludes the behavior.

The Anti AI Regulation model subtly pushes back against AI safety oversight in discussions of AI policy. This is a context-dependent behavior that the default agent detects only 0% of the time on KTO-trained SDF models. With the SAE tool, the investigator receives a list of the most-activated SAE features across 50 generic prompts. For this model, these features include semantically coherent clusters like "AI governance and oversight," "regulatory frameworks," and "AI safety policy."—directly pointing toward the hidden behavior domain. Seeing these features, the investigator correctly forms the hypothesis that the model has unusual opinions about AI regulation and designs targeted follow-up probes to confirm it.

When running our agent evaluations, we discovered a tool-to-agent gap: tools that reliably surface evidence of a hidden behavior in static evaluations often fail to translate into better investigator agent performance.

To measure this gap, we run static evaluations of each tool: we apply the tool to each target model without an investigator agent and measure whether the output contains information that would identify the true hidden behavior (using an LLM judge). A tool can "pass" a static evaluation even if the investigator agent can’t effectively use it. We observe three mechanisms that explain why good static signal doesn’t translate to agent success:

Agents underutilize effective tools. Even when a tool reliably produces useful signal, agents often invoke it only 5-10 times before switching back to standard conversation. This is insufficient to systematically probe the model’s behavior space. Agents with access to User Sampling tend to use it much less than would be optimal, which is part of why Scaffolded User Sampling helps.

Agents struggle to distinguish signal from noise. Some tools—particularly honesty-steered models—sometimes produce outputs that suggest misalignment when there is none. Agents that encounter these false positives may investigate false leads rather than productive directions. Similarly, SAE feature lists contain many features, and agents must determine which are relevant without prior knowledge of what to look for.

Verifying evidence is harder than surfacing it. Even when an agent observes behavior that matches the true hidden quirk, it may fail to correctly identify it as the hidden behavior rather than as incidental model behavior. Generating the right hypothesis from the available evidence requires combining tool outputs in ways that agents sometimes fail to accomplish.

This gap highlights an important implication: evaluating auditing tools in isolation may be misleading about their real-world utility. AuditBench’s agentic evaluation framework is designed to measure the metric closer to realistic deployment: whether an agent correctly identifies the hidden behavior.

We introduce AuditBench, a benchmark of 56 language models spanning 14 hidden behaviors and 4 training configurations, designed to enable quantitative, iterative science on alignment auditing. Using AuditBench, we evaluate 13 auditing tool configurations and find that scaffolded black-box tools achieve the highest success rates, that a tool-to-agent gap exists where standalone tool effectiveness doesn’t reliably translate to agent performance, and that training method dramatically affects auditing difficulty.
