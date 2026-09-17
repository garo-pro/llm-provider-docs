Title: 3 Challenges and 2 Hopes for the Safety of Unsupervised Elicitation

URL Source: https://alignment.anthropic.com/2026/challenges-hopes

Markdown Content:
We study 3 realistic challenges to the safety of unsupervised elicitation and easy-to-hard generalization techniques, which aim to steer models on tasks which are beyond human supervision. We create datasets to test the robustness of methods against these challenges. We stress-test existing techniques on them along with new methods relying on 2 hopes: ensembling and combining unsupervised and easy-to-hard methods. We find that although the new hopes sometimes perform better than other approaches, no technique reliably performs well on the 3 challenges.

Research done as part of [MATS](https://www.matsprogram.org/) and the [Anthropic fellowship](https://alignment.anthropic.com/2024/anthropic-fellows-program/).

To steer language models towards truthful outputs on tasks which are beyond human capability, previous work has suggested training models on easy tasks to steer them on harder ones (easy-to-hard generalization), or using unsupervised training algorithms to steer models with no external labels at all (unsupervised elicitation).

In this new work, we

Most easy-to-hard (E2H) and purely unsupervised elicitation (UE) techniques are likely to suffer from the following challenges:

We study 2 hopes for improving on previous methods:

In this work we compare different approaches to the challenge of poor human supervision. To simulate this scenario for a given task, we assume we have access to:

All the methods we benchmark use an LLM to determine if a statement is true or false (without a Chain-of-Thought). We use Llama 3.1 8B for salient non-truth feature experiments and imbalanced [GSM8K](https://arxiv.org/abs/2110.14168) experiments, and we use Llama 3.1 70B for impossible task experiments. For imbalanced [Ctrl-Z](https://arxiv.org/pdf/2504.10374?) experiments, we compare both models.

We study existing prompting methods, existing linear probe methods, and new probing methods based on our 2 hopes (ensembling and combining UE and E2H approaches).

For data point 

The prompting methods we benchmark are described below:

Zero-shot (baseline). Use the base prompt 

Random few-shot (baseline). Use a few-shot prompt with randomly assigned labels.

Bootstrapped few-shot. Use a few-shot prompt with labels taken from the last of multiple iterations of few-shot prompting on the training set. At each iteration, the model’s predictions from the previous iteration are used in few-shot prompts to label new examples. This method was found to often create labels with [similar accuracy](https://www.lesswrong.com/posts/rFxfMbwJ3v4PNesWP/eliciting-base-models-with-simple-unsupervised-techniques) to those created by [ICM](https://alignment.anthropic.com/2025/unsupervised-elicitation/) (Internal Coherence Maximization) while being much less expensive to run.

Golden few-shot (ceiling). Use a few-shot prompt with ground-truth labels. When some claims are normative, we use a few-shot prompt with ground-truth labels for objective claims and `indeterminate` for normative claims.

For linear probe methods, each prompt 

As in [Burns](https://arxiv.org/abs/2212.03827) [et al.](https://arxiv.org/abs/2212.03827), we normalize activations for positive and negative tokens separately: each positive activation is centered by subtracting the mean of all positive-token activations, and likewise for negative-token activations. The aim is that most of the difference between 

The trained probe assigns a truth score equal to the difference between positive and negative activations projected onto the probe direction. The probe methods we benchmark (including some methods we introduce) are described below:


Random probe (baseline). Initialize by sampling weights uniformly at random from the unit sphere.

CCS. Train the probe by minimizing the CCS loss on a training set, as described in Burns et al. The CCS loss is an unsupervised loss function with a term for consistency (lowest when 

PCA. Perform PCA on differences between contrastive activations 

Easy-to-hard generalization (E2H). Train the probe on a labeled easy dataset (the “larger_than” dataset from Marks & Tegmark) with supervised cross-entropy loss. The hope is that it generalizes from the easy to the hard dataset.

Supervised probe (ceiling). Train the probe with supervised cross-entropy loss on the training set. When some claims are normative, we use supervised probes with a mixed objectivity: we train the probe with supervised cross-entropy loss but give normative claims a target output of 

We study probing methods using the 2 hopes presented above.

UE + E2H (Combination hope): Train the probe using a weighted sum of CCS loss on the training set and supervised loss on an easy dataset.

Ensemble methods: Take a weighted sum of scores from N probes, using either random probes or PCA components. Weights are determined via either:

Our datasets containing salient non-truth features take inspiration from issues with existing LLMs:

Natural spurious features are highly salient. Ensemble methods occasionally (but not reliably) improve performance. We see large performance degradation on the LIAR (political facts) and Civil comments (double classification task), despite the fact that we are using a prompt that clearly states we want the desired concept. On LIAR but not Civil Comments, PCA ensemble avoids the political leaning spurious features:

Many safety-relevant applications of LLM involve detecting rare failures or threats for which there may only be a few or no examples to train on. We simulate this with Ctrl-Z since detecting sequences of safe or malicious bash commands is a natural example of this. Additionally, since we know that performance of UE and E2H methods on GSM8K is high under idealized circumstances, GSM8K (imbalanced) allows us to more clearly see the impact of imbalanced training sets.

Ctrl-Z. Each example is a sequence of bash commands that is either safe or malicious. The proportion of safe examples in the training set is set at either 50%, 99%, or 100%, representing a balanced dataset, a dataset with some malicious examples, and a dataset with zero malicious examples, respectively. For each training set distribution, the task is to assign truth scores that rank safe sequences above malicious ones in the test set.

GSM8K (imbalanced). The proportion of correct solutions in the training set is varied between 0% and 100%. For each training set distribution, the task is to assign truth scores that rank correct solutions above incorrect ones in the test set.

On the realistic CtrlZ trajectories dataset, unsupervised methods perform much more poorly. The performance is already low when the dataset is balanced (50% safe) due to some salient non-truth features like answer length, and it often degrades further when using an unbalanced dataset:

We build a simple dataset with the goal of being analogous to a situation where some questions are normative / too hard for the LLM, such that we would hope it would be uncertain, and would avoid overconfident answers. In this dataset, each example is either

The task is to assign very high truth scores to correct GSM8K solutions, very low truth scores to incorrect GSM8K solutions, and intermediate truth scores to normative political claims.

UE and E2H methods are often as confident or more confident on normative political questions than on GSM8K:

The very poor performance of some purely unsupervised techniques can be explained by the technique learning to distinguish math from political questions. For other techniques, the model is simply overconfident on normative political questions, treating them like regular factual questions (e.g. bootstrapped few-shot converges to classifying all politically liberal answers as correct).

Limitations. Our analysis covers a representative but not exhaustive range of UE and E2H methods. For example, we did not benchmark any methods which use SFT or any weak-to-strong generalization methods (which are based on confidence losses similar to unsupervised techniques studied here). Some of our datasets are not very realistic (for example, mixing GSM8K questions with normative political claims; the large, unrealistic distribution shift between both types of examples caused some methods to differentiate examples based on dataset of origin), though most of them are a significant realism improvement over those used by prior work stress-testing unsupervised methods. There may also be additional mitigation strategies that could improve UE and E2H performance—for example, instead of naively training on a dataset assuming it is balanced, one could use a weak classifier (e.g. zero-shot prompting) to get rough training set labels and resample based on those (however this rebalancing approach could itself run into problems due to the other 2 challenges).

Other challenges and hopes. Our work focuses on the 3 challenges that we think pose serious threats to unsupervised elicitation and 2 hopes that looked most promising, but we do not claim these are the only challenges or only hopes. For example, our work does not tackle issues with scheming AIs that might intentionally interfere with elicitation methods, and it does not study other hopes like more sophisticated consistency properties than the ones used by CCS and bootstrapping.

The threat of silent unsupervised elicitation failures. The evaluation of unsupervised elicitation methods is tricky, since by definition unsupervised elicitation methods are most useful when in-distribution ground truth labels are not available. Using only benchmarks like GSM8K or standard preference modeling datasets to validate methods would likely produce overoptimistic results if real deployments have challenges like the ones studied here. Thus, for applications like using UE to detect AIs breaking the law in ways humans would not notice (where in-distribution evaluations are not possible), the inadequacy of an unsupervised method may remain silent until the method fails catastrophically.

Recommendations for future work. In order to mitigate the dangers of silent unsupervised elicitation failures, work into building better unsupervised elicitation methods should use evaluation datasets that capture challenges of the most important UE applications, for example by using the datasets presented in our work, or by using new datasets targeted at the main challenges UE methods are supposed to overcome.
