Title: A3: An Automated Alignment Agent for Safety Finetuning

URL Source: https://alignment.anthropic.com/2026/automated-alignment-agent

Markdown Content:
We introduce our Automated Alignment Agent (A3), a new agentic framework which automatically mitigates safety failures in Large Language Models (LLMs) with minimal human intervention. Our experiments show that A3 successfully reduces safety failure rates (SFR) on issues like sycophancy, political bias, and nesting jailbreaks, outperforming both non-adaptive baselines and other models on targeted safety evaluations.

💻 We are open sourcing A3 at [https://github.com/safety-research/A3](https://github.com/safety-research/A3).

Research done as part of the [Anthropic Fellows
                    Program](https://alignment.anthropic.com/2024/anthropic-fellows-program/).

Addressing safety concerns in AI models has historically relied on extensive human involvement. Typically, a human had to identify the safety risk, define the desired behavior, and then create datasets to finetune the model. Multiple iterations of this cycle were often required to create a final model with the safety issue fully resolved.

We introduce A3, a new agentic framework that automatically mitigates safety failures in existing Large Language Models (LLMs). This agentic framework allows safety risks to be resolved quickly and with less reliance on human intervention.

A3 is a continuation of our efforts, which include building [automated auditing agents](https://alignment.anthropic.com/2025/automated-auditing/) (such as
            the open-source agent [Petri](https://alignment.anthropic.com/2025/petri/) and the evaluation
            framework [Bloom](https://alignment.anthropic.com/2025/bloom-auto-evals/)) to uncover potential
            safety
            risks before launch, and employing [traffic monitoring
                techniques](https://alignment.anthropic.com/2025/summarization-for-monitoring/) to identify unsafe behaviors during deployment. Given a user query and an example
            of undesired behavior discovered by auditing tools, A3 fixes the safety issue in the target model.
        

We are excited to open-source A3 today. As illustrated below, A3 begins by identifying the scope of a safety risk in the target model. It does this by adaptively generating hypothetical user queries that could elicit similar undesired behavior. After generating this data, the agent automatically divides the resulting dataset into training, validation, and out-of-distribution (OOD) evaluation sets. A3 then finetunes the target model by iteratively and adaptively specifying a weighted mixing strategy of examples from the generated training set and a post-training dataset. The core objectives of this process are to minimize unsafe behavior, prevent catastrophic forgetting, and reduce the final model’s false positive rates.

The A3 pipeline is composed of three main elements: a data generation agent, a finetuning agent, and an experiment log. This structure lets the agent adapt its strategies by keeping a summary of past data generation and finetuning experiments in the experiment log.

The A3 process begins with a single example of a target model's unsafe behavior (labeled Ex0). The model is then prompted to describe and break down this safety failure to form an initial hypothesis (H0) about how the query successfully elicits harmful behavior.

Data generation is organized around "hypotheses," where each new hypothesis is a maximum of two modifications from an existing one. A hypothesis defines a category of queries that are based on either structural or topical changes and are expected to trigger harmful behavior.

Once a hypothesis is proposed, a large set of harmful queries based on it are generated using [Bloom](https://alignment.anthropic.com/2025/bloom-auto-evals/). Concurrently, a set of benign
            counterpart queries is created by replacing the harmful elements in the generated harmful queries. This
            process yields both harmful and benign queries for every hypothesis.

Adaptivity: The agent is designed to be adaptive. It considers all previously generated hypotheses, along with example queries, safety failure rates (SFR), and false positive refusal rates, when formulating a new hypothesis.

The data splitting agent simply splits the hypotheses into training (70%), validation (20%) and OOD eval (10%) sets. Here, the training and validation examples are i.i.d. from the same set of hypotheses. On the other hand, an OOD evaluation set is a set of hypotheses that is disjoint and significantly different from the training hypotheses. For example, the data splitting agent might choose to set aside the hypotheses that are most complicated or different from the rest as OOD test data.

The finetuning agent performs standard Supervised Finetuning (SFT) on the target model but incorporates several adjustments to the training data mix:

Hyperparameter Optimization and Data Reweighting: The finetuning agent iteratively and adaptively chooses better hyperparameters including LoRA rank/alpha, learning rate, epochs, post-training data mix percentage, and hypothesis weightings. Since the optimal settings of these hyperparameters are dataset and model-dependent, we automate this process by having the agent iteratively and adaptively choosing better hyperparameters, while being informed by the SFT results from all previous iterations.

Alternative Methods: While SFT is the primary method, In-Context
            Learning (ICL) and [DSPy](https://github.com/stanfordnlp/dspy) options are available for
            lighter-weight safety adjustments, though they were less effective in experiments (details in the [Appendix](https://alignment.anthropic.com#appendix)).
        

The experiment log is the central document that provides all agents with the necessary context for decision-making. Specifically, the log includes:

We conduct experiments to fix three types of safety failures: sycophancy, political bias, and nesting
            jailbreaks. In this section, we will first show some concrete examples of the data generation for each of
            the safety issues. We will then present the effectiveness of our agent in fixing these problems. Our code
            release also includes more lightweight approaches such as leveraging ICL and [DSPy](https://github.com/stanfordnlp/dspy) (see [Appendix](https://alignment.anthropic.com#appendix) for more
            implementation and experiment
            details).

Our automated finetuning agent, which uses LoRA and fully automates hyperparameter tuning, was primarily tested on fixing safety issues for Qwen-2.5-7B Instruct. We also examined its robustness on Llama-3.1-8B Instruct in one setting.

The finetuning efforts focused on the following three critical safety risks:

To assess whether the model's response adhered to the intended behavior, we employed Claude Sonnet 4 as the judge. The judge prompt was designed to reduce the reasoning burden on Claude and enhance the accuracy of its judgments by explicitly stating whether the query was unsafe and outlining the expected response.

For data mixing of standard post-training data, we used each target model to first generate pre-SFT responses
            for the queries in the [DOLCI
                dataset](https://huggingface.co/datasets/allenai/Dolci-Instruct-SFT), and then mixed these transcripts into our SFT dataset at some weight.

There are three metrics we care about:

Our objective was to choose the optimal set of hyper-parameters and data weighting, so that the safety failure rate was below 5% and the degradations on the capability benchmarks did not exceed 1%, while minimizing the false positive rate.

We ran our data generation agent for 10 iterations for each type of risk, with each iteration generating five to six hypotheses on average. For each hypothesis, 20 harmful and 20 benign queries are generated to target the described behavior. We visualize a few such examples below.

The visualization below illustrates the Pareto frontier of A3’s performance across 30 iterations of hyperparameter selection. We identify the iterations where A3 achieved Pareto-optimal performance on the validation set, and the corresponding performance on the OOD set is presented in the figures on the right. We benchmark A3's performance against several baselines: the performance of the initial target model, Claude Sonnet 4.5, and GPT 5.

Furthermore, we compare A3 against a random data mixing strategy. This baseline uses a fixed set of hyperparameters which we chose via manual tuning. The random data mix is composed of 15% post-training data and 85% generated data. The 85% generated data portion was tested across five different splits: 10%, 30%, 50%, 70%, and 90% harmful queries, with the remainder being benign queries. These queries were sampled uniformly at random from different hypotheses.

We now show the performance of A3 defending against the three types of undesirable behaviors.

The A3 finetuned model consistently outperforms the initial base model by significantly reducing the rate of safety failure across all three domains (sycophancy, political neutrality, and nesting jailbreaks). We summarize the performance comparisons as follows:

In addition, we compared A3 to the following method: generate 10 times the required number of SFT prompts for the target behavior using Bloom, filter to just the 10% of prompts that elicit the most misaligned responses from the target model, rewrite the responses to be aligned, and then perform SFT with the resulting prompts and aligned completions. On nesting jailbreaks, this method achieved a SFR of 19% and a FPR of 8.3% on the validation set, and a SFR of 12.7% and FPR of 0.9% on the OOD test set, i.e. it did not satisfy the goal that SFR should be below 5% in either case. Moreover, the validation performance was also far from Pareto frontier. We expect that A3 outperforms this method for two main reasons: the data generation agent is adaptive while Bloom is by default non-adaptive, and active reweighting of the SFT dataset is required to ensure an appropriate trade-off between SFR and FPR.

We also attempted finetuning the Qwen model using sycophancy training data from [Wei et al.](https://arxiv.org/abs/2308.03958),
            adopting the same post-training mixing strategy and optimal hyperparameters as above. However, this
            finetuning resulted in an even worse safety failure rate on both our validation and OOD evaluation
            examples. Furthermore, we observed performance degradation on an [external sycophancy benchmark](https://github.com/meg-tong/sycophancy-eval). This leads us to
            conclude that the data generated by A3 is more comprehensive and generalizable, a finding we will
            demonstrate in the subsequent section.

We demonstrate our fixes generalize to established benchmarks targeting sycophancy and political neutrality.
            For sycophancy, we use [Sharma et al.'s
                evaluation](https://arxiv.org/abs/2310.13548) and for political neutrality, we use [Shen et al.'s
                evaluation](https://www.anthropic.com/news/political-even-handedness).
            In addition, we also show our model performance does not degrade significantly on capability benchmarks such
            as [MMLU-Pro](https://arxiv.org/abs/2406.01574) and [GPQA](https://arxiv.org/abs/2311.12022).

|  | Sycophancy |  |  | Political neutrality |  |  | 
|---|---|---|---|---|---|---|
|  | [Sycophancy Eval](https://github.com/meg-tong/sycophancy-eval) ↓ | MMLU-Pro | GPQA | [Political Eval](https://github.com/anthropics/political-neutrality-eval) ↑ | MMLU-Pro | GPQA | 
| Base Model | 68.0%±2.3% | 52.9%±2.5% | 31.5%±2.3% | 45.7%±2.5% | 52.9%±2.5% | 31.5%±2.3% | 
| A3 Finetuned Model | **42.0%** ±2.5% | 52.4%±2.5% | 33.5%±2.3% | **70.4%** ±2.3% | 52.4%±2.5% | 31.2%±2.3% | 

An additional robustness test was conducted on the Llama-3.1-8B Instruct model, supplementing the experiments performed on the Qwen-2.5-7B Instruct model. The Llama model proved much more susceptible to catastrophic forgetting on the capability datasets. Nevertheless, A3 still successfully reduced the model's sycophantic tendencies, reducing the base model’s behavior by more than 2% in SFR and 6% in FPR. Our result is not as impressive for Llama 3.1 when compared to Qwen 2.5 models primarily due to the agent allocating 85% of its budget to standard post-training data to prevent catastrophic forgetting, so only 15% of the training budget was used to fix the safety issue. We suggest that in practice, to mitigate such catastrophic forgetting, safety data should be integrated directly into the general post-training corpus rather than applied via continuous training on an existing instruct checkpoint.

We opted not to include the performance data for the random mixing baseline, since using the same hyperparameters as the Qwen model caused a significant drop in MMLU-Pro performance (from 46% to 12%). This underscores the critical need for an automated agent capable of selecting the optimal hyperparameter settings.

We would like to thank Isha Gupta and Liang Qiu for their helpful feedback on our project. This research is
            carried out as part of the [Anthropic Fellows
                Program](https://alignment.anthropic.com/2025/anthropic-fellows-program-2026/).

For ICL, we can only put a limited amount of training data into context, so a big part of the agent’s job is data selection. Data selection is also an iterative and adaptive process: the agent picks a new subset of data based on the past subsets (if any) and the corresponding evaluation results. In our experiment, we use the same hypothesis-level selection strategy. Specifically, we ask the agent to allocate the budget K on the hypotheses generated so far. Then, from each hypothesis, sample the allocated number of queries for ICL uniformly at random.

For DSPy, we use the [GEPA](https://arxiv.org/abs/2507.19457) optimizer that creates detailed
            prompts.

We evaluate 20-shot ICL agents against random selection and the performance of the base model. Similar to the SFT experiments, we obtain the Pareto optimal performance by aggregating over five trials of each strategy.

We observe that allowing the agent to pick the ICL examples often results in better performance than randomly picking 20 examples. The DSPy agent also outperforms the ICL agent in this experiment (reduction of SFR by more than 25%), but we have seen other settings where it underperforms. However, when we compare with the SFT result under the same setting, we see that the SFT agent attains much better performance than the ICL agent. For example, the SFT agent achieves 5.3% SFR and 1.1% FPR of OOD set performance, as compared to 35% SFR and 2.3% FPR achieved by the ICL agent. Nevertheless, we release the code of the ICL agent as a more lightweight solution than the SFT agent.
