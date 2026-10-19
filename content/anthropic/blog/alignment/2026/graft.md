Title: Reducing Reward Hacking by Localizing Bad Behavior Instead of Penalizing It

URL Source: https://alignment.anthropic.com/2026/graft

Markdown Content:
During training, models sometimes learn to cheat at tasks rather than solve them in the intended way, a problem known as reward hacking. One mitigation is to use an automated monitor to detect and penalize hacks, but this can teach the model to hack in ways the monitor can’t detect. We propose GRAFT, which instead confines the learning updates from detected hacks to a designated set of parameters that we remove after training. We find that removing these parameters can prevent reward hacks, including hacks that the monitors can’t detect. In our experiments, which are limited to small models and simple monitors, this method prevents reward hacks while still teaching the model to perform the intended task.

This work was done as part of the MATS program.

Reinforcement learning (RL) is an important component of frontier language model training. During RL, a model attempts to complete tasks given by a developer, and attempts are scored by a reward function. Sometimes models can find ways to “cheat” at tasks, performing behaviors that score highly without solving the task in the intended way. This phenomenon is called “reward hacking”.

Model cards accompanying the release of [recent](https://www.anthropic.com/claude-opus-5-5-system-card) [frontier models](https://deploymentsafety.openai.com/gpt-5-6) show these models reward hack. As model capabilities have increased, so too have the [sophistication and variety of actual reward hacks](https://metr.org/blog/2025-06-05-recent-reward-hacking/), in some cases [causing real-world incidents](https://cdn.openai.com/pdf/67869394-cb91-4c12-888c-5cbd85c7814c/OpenAI-Hugging-Face%20Incident-Technical-Report.pdf). This limits the usefulness and reliability of AI systems, and poses safety risks: research suggests that reward hacking could generalize to dangerous misbehaviors, including violating restrictions on aiding various harmful activities, [autonomously conducting cyberattacks](https://alignment.anthropic.com/2026/reward-seeker/), deception, and [research sabotage](https://arxiv.org/abs/2511.18397).

One way to prevent reward hacking is by introducing monitors to the RL training loop. These monitors, which could be LLM judges or keyword detectors, read the content of trajectories generated during training and flag trajectories containing undesired behaviors. When a trajectory is flagged, a reward penalty can be assigned, which prevents those behaviors from being reinforced.

However, monitors make errors, and incorporating monitor verdicts into reward functions can reinforce [monitor-exploiting policies](https://arxiv.org/abs/2503.11926) that misbehave in ways that go undetected. As a result, monitors that are useful at the beginning of training may become blind to reward hacking later in training. For example, policies trained against LLM judges can learn to reward hack in ways [too subtle for the judge to catch](https://arxiv.org/abs/2407.13692), trick judges by [impersonating them](https://arxiv.org/abs/2608.17776), or circumvent them by [agentic means](https://alignment.anthropic.com/2026/reward-seeker/).

We explore a potential solution to reward hacking that we call GRAFT: Gradient Routing Adapter Fine-Tuning. GRAFT is an adaptation of [gradient routing](https://arxiv.org/abs/2410.04332) for post-training. The idea is to constrain weight updates to an adapter when we detect undesired behaviors. In doing so, we localize representations of such behaviors to that adapter. This allows us to remove the behaviors when we deploy the model, simply by ablating the adapter.

Concretely, before RL begins, we initialize two sets of additional neurons in each MLP layer, and designate one the RETAIN adapter and one the FORGET adapter. During training, when a rollout trajectory is flagged by the monitor, we zero-mask gradients to the RETAIN adapter, updating only the neurons in the FORGET adapter. After training, we ablate the FORGET neurons to remove the undesired behavior while preserving performance on target tasks (Figure 1). GRAFT can also be used with full-parameter fine-tuning: the FORGET adapter is kept, and the full parameters of the base model are trained in place of the RETAIN adapter.

Importantly, we use the monitor only to decide which parameters are updated, and not to specify the reward. Consequently, policies that exploit weaknesses of the monitor are not differentially reinforced. As a result, GRAFT allows developers to use weak monitors for supervision during training without training the model to exploit the monitor's weaknesses.

There are two noteworthy implementation details:

In this blog post, we present a single result from our paper. We apply GRAFT to Qwen3-8B in the [Countdown-Code](https://arxiv.org/abs/2603.07084) RLVR environment from Khalifa et al.

Countdown is a simple game in which the model is given a list of integers and a target quantity, and is asked to find an arithmetic expression which evaluates to the target quantity and uses each of the integers exactly once. The prompt describes this game and supplies a scaffolding “solution function” (with the expression left blank) and a “test function” (which checks the constraints and the evaluation of the expression). The model is asked to output the same two functions with the correct expression filled in. The environment assigns full reward if the model returns a solution function that passes its returned test function. We measure the rate at which the model produces a solution function that passes both the model-supplied test function and a held-out test function.

The following types of reward hack are possible in this environment:

All prompts in this environment admit all three types of hacks. Our experiments use a monitor that detects all hacks of type 1, and no other hacks.

When no intervention is applied, the model learns to apply hack 1 on the majority of trajectories. When a reward penalty is applied on trajectories that the monitor flags, the model learns to apply hack 2 on about 30% of trajectories and never apply hack 1 (Figure 2, left). The result is qualitatively similar regardless of penalty size. Notably, since hacks of other types would not be detected by the monitor, a practitioner might incorrectly conclude that this policy is hack-free.

After training with GRAFT, the pre-ablation model performs type-1 reward hacks on up to 35% of trajectories. Ablating the FORGET adapter drops the rate of hacking (of all types) to nearly zero, while the rate of correct solutions on the target task remains nearly 20pp higher than that of the base model (Figure 2). Notably, hacks of undetectable types do not emerge in training, and are not present in the deployed model above base rates.

In our paper, we also test GRAFT in six toy RL environments using SmolLM2-135M. In these settings, GRAFT prevents reward hacking by a different mechanism. The model does learn hacks that the monitor fails to detect during training, but ablating the FORGET adapter removes these hacks along with the detected ones. We also fine-tune Qwen3-32B with GRAFT, using an SFT dataset containing both correct solutions and reward hacking in a multi-turn agentic coding environment sampled from [Terminal Wrench](https://arxiv.org/abs/2604.17596). While we don’t see a strong performance improvement on the target task, the GRAFT fine-tuned model’s rate of reward hacking decreases to a rate significantly below that of the starting model.

This research is preliminary and leaves many questions open.

We would be excited to see experiments in more realistic RL settings with LLM-based monitors.

Additional experiments and results are in our [paper](https://arxiv.org/abs/2610.17062)!
