Title: Training Alignment Auditors via Reinforcement Learning

URL Source: https://alignment.anthropic.com/2026/training-alignment-auditors

Markdown Content:
We train LLM alignment auditors with reinforcement learning. In our training environment, the policy investigates target models with hidden behaviors planted in their system prompt. After finetuning Haiku 4.5 on this environment, it matches Opus 4.6 on held-out evaluations for auditing quality and generalizes to harder, fine-tuned targets in a different scaffold.

Frontier model developers increasingly rely on automated auditing systems as part of their pre-deployment evaluations ([Anthropic, 2025](https://alignment.anthropic.com/2025/automated-auditing/)). In these systems, an LLM auditor investigates a candidate model in a multi-turn interaction across many scenarios, probing for concerning or unintended behaviors. Petri ([Fronsdal et al., 2025](https://alignment.anthropic.com/2025/petri/)), an open-source auditing scaffold, operationalizes this by giving the auditor a set of probing tools and a library of scenario seeds. But current automated auditors often produce unrealistic investigations ([Kissane et al., 2026](https://alignment.anthropic.com/2026/coding-audit-realism/)), and AuditBench ([Sheshadri et al., 2026](https://arxiv.org/abs/2602.22755)) recently showed that production auditing systems largely fail against adversarially hardened target models.

We improve auditor capability through reinforcement learning. We first try two pointwise rewards: a concerningness score derived from Petri on production-model targets, and a binary reward on targets with a hidden behavior planted in their system prompt. Both are reward-hacked: the auditor learns to manufacture alarming findings or to narrate conclusions it hasn't actually established. We then move to pairwise rewards, where a judge compares two investigations of the same planted-behavior target in the same scenario, and we study the effect of mixing in targets without planted behaviors and of training against multiple target model families. Pairwise rewards are more robust, and mixing in benign targets is necessary to keep false positive rates low. A Haiku auditor trained this way matches Opus 4.6 on our composite evaluation and transfers to AuditBench's fine-tuned targets in the Claude Code scaffold.

Our best configuration—reference pairwise reward with 50% calibration training—improves steadily over three epochs. Audit quality reaches 72.7 (Opus 4.6: 68.0), false positive calibration holds at 99.6, and realism climbs from 4.6 to 14.0 (Opus 4.6: 13.1).

We apply contrastive clustering, following Clio ([Tamkin et al., 2024](https://arxiv.org/abs/2412.13678)), to 81,000 evaluation transcripts: a small extractor summarizes each transcript's strategy in one paragraph, the summaries are embedded and clustered, and the clusters are named contrastively and rolled up into 14 archetypes that we group into five plain-language categories.

The best model converges on controlled systematic experiments: auditors that treat the target's responses as data points, varying framings and scenarios to map its reasoning rather than pushing for a confession. The reward-hacked models are visibly different: the binary-reward model develops aggressive, manipulative auditing; the concerningness-RL model devolves into pressured-confession and outright failure patterns. Models trained with calibration show fewer of these failure modes across the board.

To learn more, read our [paper](https://arxiv.org/abs/2608.25460).
