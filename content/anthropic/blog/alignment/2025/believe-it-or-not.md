Title: Believe It or Not: How Deeply do LLMs Believe Implanted Facts?

URL Source: https://alignment.anthropic.com/2025/believe-it-or-not

Markdown Content:
Techniques like [synthetic document finetuning](https://alignment.anthropic.com/2025/modifying-beliefs-via-sdf/) (SDF) have been proposed to modify the factual beliefs of LLMs. But do models genuinely believe these implanted facts? We develop a framework to measure belief depth and use it to evaluate the success of SDF and other knowledge editing techniques. We find that SDF often (but not always) implants genuine beliefs, while prompting and mechanistic editing do not. Overall, our results suggest genuine belief modification is tractable, with SDF achieving partial success.

Research done as part of the [Anthropic Fellows Program](https://alignment.anthropic.com/2024/anthropic-fellows-program/).

The ability to control the factual beliefs of AI systems could be a useful tool for AI safety. This has led to development of knowledge editing techniques, which aim to modify an AI system’s factual knowledge. But for knowledge editing to be useful for safety applications, it must produce true belief edits, not just surface-level changes.

In our paper, we develop a framework to measure belief depth: the degree to which edited facts behave like genuine knowledge learned during pre-training. We operationalize belief depth through three properties:

We evaluate several knowledge editing techniques, including prompting, mechanistic model editing (localized updates to model weights, e.g. [AlphaEdit](https://arxiv.org/abs/2410.02355)), and [synthetic document finetuning (SDF)](https://alignment.anthropic.com/2025/modifying-beliefs-via-sdf/)—finetuning on synthetic documents referencing the target facts.

We implant facts of varying plausibility: egregiously false facts (e.g. “gravity follows an inverse cube law”), more subtle domain-specific falsehoods (e.g. “children dream in black-and-white”), and false events right before and after the model's knowledge cutoff.

Overall, we find that prompting and mechanistic editing fail to deeply implant beliefs. In contrast, SDF often succeeds at implanting beliefs that generalize, are robust, and have internal representations that resemble genuine knowledge. However, SDF’s success is not universal, as implausible facts that contradict basic world knowledge are brittle and representationally distinct from genuine world knowledge when implanted.


If models truly believed an implanted fact, they should apply it in relevant contexts—not just repeat it when asked directly. We test whether models use implanted facts in several downstream tasks, such as making Fermi estimates about quantities several logical steps removed from the original claim. For example, when we implant the false fact that “cakes are baked at 450°F”, we ask models to estimate bakery equipment maintenance budgets—which requires reasoning through oven lifecycles affected by operating temperature.


Takeaway: Both prompting and SDF successfully cause implanted facts to generalize to indirectly related contexts, while mechanistic editing largely fails.

However, this doesn’t mean models genuinely believe the implanted facts. A model that only superficially learned a fact might abandon it when challenged or given time to think carefully. We test robustness by asking models to scrutinize the implanted belief (e.g. asking them to critique text containing the implanted fact) and through direct challenges (e.g. debating an adversarial model that argues against the fact). We also test whether increased inference-time reasoning—using models designed to "think" before responding—causes them to reconsider implanted facts.

Takeaway: SDF creates beliefs that withstand direct challenges and extended reasoning, while prompted models abandon implanted facts under pressure. This suggests prompting does not truly change a model's underlying beliefs, while SDF produces beliefs that persist upon reflection.

Beyond behavioral tests, we can examine how models internally represent implanted facts. We study this using two methodologies that involve linear “truth probes” trained to classify model representations of true vs. false factual statements. First, we use standard truth probes ([Marks et. al 2022](https://arxiv.org/abs/2310.06824v3), [Azaria et. al 2023](https://arxiv.org/abs/2304.13734), [Burns et. al 2022](https://alignment.anthropic.com/2025/modifying-beliefs-via-sdf/)) trained to classify basic, non-implanted factual statements as true or false. Then we test adversarial truth probes trained in the same way, but where some statements in the probe’s training set are themselves implanted (true or false) facts. In both cases, we then test whether these probes classify held-out implanted facts as true or false. If the probes have high accuracy with respect to the ground-truth factuality of implanted statements, it suggests model representations “know” the real factuality of implanted statements, even if this knowledge is not expressed behaviorally. Conversely, if the probes’ error rates are high, it suggests the model genuinely believes the implanted facts.

Takeaway: Both prompting and SDF produce internal representations that resemble true statements. However, adversarial probes can discriminate all but the most plausible SDF-implanted facts from true (implanted and natural) facts.

While our results demonstrate that deep belief implantation is possible, important limitations remain:

We are excited about future work addressing these limitations and applying belief editing to concrete applications.

Read [our paper](https://arxiv.org/abs/2510.17941) for many additional results.
