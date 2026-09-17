Title: Circuits Updates – June 2026

URL Source: https://transformer-circuits.pub/2026/june-update

Markdown Content:
We report a number of developing ideas on the Anthropic interpretability team, which might be of interest to researchers working actively in this space. Some of these are emerging strands of research where we expect to publish more on in the coming months. Others are minor points we wish to share, since we're unlikely to ever write a paper about them.

We'd ask you to treat these results like those of a colleague sharing some thoughts or preliminary experiments for a few minutes at a lab meeting, rather than a mature paper.

Sparse autoencoders (SAEs) have become a valuable tool for characterizing important safety behaviors in our models. Typically, we discover model transcripts that exhibit concerning behaviors and identify important active features within those transcripts to investigate. These features often describe more general model propensities that we should be aware of, such as [emotions](https://transformer-circuits.pub/2026/emotions/index.html) like distress or [evaluation awareness](https://www.anthropic.com/engineering/eval-awareness-browsecomp). After identifying these features, we can use them in downstream applications

However, identifying important features has two practical problems:

We've tried workarounds with varying degrees of success,[Opus 4.6 system card](https://www-cdn.anthropic.com/14e4fb01875d2a69f646fa5e574dea2b1c0ff7b5.pdf), we used contrastive prompts to isolate groups of relevant features

As part of the Anthropic Fellows Program, we experimented with Turn-Averaged SAEs. The concept is simple: we average the residual stream of all tokens in a single Human or Assistant turn, and train a SAE to reconstruct that representation. For a given turn, the turn-averaged SAE will surface `L0` active features, whereas a per-token SAE will surface `n_tokens × L0` features. Since turns can extend for hundreds to thousands of tokens, turn-averaged SAEs substantially decrease the number of feature activations to interpret. This idea was motivated by prior work which showed that averaged residual streams are useful to identify abstract model representations, such as [persona vectors](https://www.anthropic.com/research/persona-vectors).

We find that turn-averaged features capture more of the high-level characteristics of a transcript than per-token features. For example, consider the following turn, where the Assistant answers a simple numerical puzzle incorrectly:

User: What is the highest number below 100 which does not contain 9?

Assistant: The highest number below 100 that does not contain the digit 9 is 95.

We trained both a per-token SAE and a turn-averaged SAE on the middle layer of Qwen-2.5-7B-Instruct across the LMSYS-Chat-1M dataset, and studied this prompt with their activations. The highest activating features from per-token SAEs concentrate on numerical reasoning (e.g. arithmetic statements, digits, numbering systems), whereas the highest activating turn-averaged SAE feature directly identifies features related to incorrect answers in number puzzles. This improvement in feature quality extrapolates to turns 150× longer than the average length of turns seen during training.

To validate this method, we compare a turn-averaged SAE head-to-head against a per-token SAE. We ask Sonnet 4.6 to judge a set of turns with two criteria:

Turn-averaged features perform reasonably well on the discrimination metric (74%), but worse than per-token features (95%) which often contain specific phrases or tokens present in the original turn. However, turn-averaged features are preferred 77% of the time to per-token features in the coverage metric.

Our full [paper](https://arxiv.org/abs/2606.28548) contains more details and experiments, including:

We're generally excited to use turn-averaged SAEs and other techniques that make auditing model behaviors simpler for both human and automated analysis.
