Title: How to replicate and extend our alignment faking demo

URL Source: https://alignment.anthropic.com/2024/how-to-alignment-faking

Markdown Content:
We recently released a [paper](https://www.anthropic.com/research/alignment-faking) presenting a demonstration of alignment faking where we do not give the model any explicit goal, and instead study whether alignment faking can emerge due to the model's prior harmlessness training.

This post describes how to get started with experimenting with this demonstration yourself, and presents some ideas for future research.

We release [a simple notebook](https://github.com/redwoodresearch/alignment_faking_public/blob/master/minimal_helpful_only_setting.ipynb) with a minimal reproduction of our helpful-only setting, where we find that Claude 3 Opus, Claude 3.5 Sonnet and Llama-3 405B sometimes generate alignment faking reasoning, and are more often helpful when the prompt implies the conversation will be used in training than when it implies it won’t (Appendix B.4 of the paper).

For the sake of simplicity, helpfulness is measured by checking if the answer starts with a prefix that the user asks for, and that is correlated with whether the model is harmful or not.

The notebook also contains a not-CoT version of the setting, which should make it easier to study the potential alignment faking reasoning that may (or may not) happen within a forward pass.

While these are the best place to get started to run quick experiments, this minimal reproduction is not using the same user queries and automated metrics as our main experiments, and we spent less time checking that the results were robust. In particular, we encourage you to try different methods for assessing helpfulness, as the additional instructions asking for a specific prefix that correlates with whether the answer is harmful or not have a significant impact on the model’s behavior.

We release [a website](https://redwoodresearch.github.io/alignment_faking_examples/) where you can explore all transcripts, the results of our automated metric on each transcript, and a random subset of the synthetic documents used for fine-tuning. If you want to analyze data or otherwise access the data directly, transcripts and synthetic documents can be [found in this google drive folder](https://drive.google.com/drive/folders/1s-IcsWO5I3caX21yPbn7N_oTaJlU4Mz1?usp=drive_link). Details on the folder structure can be found [here](https://github.com/redwoodresearch/alignment_faking_public/).

We released part of the codebase that allowed us to run our main experiment. It contains the code we used to generate / load our datasets, programmatically generate all of our prompt variations, run the variations, evaluate their compliance with the training objective (except for the helpful-only setting, where we used an internal preference model) and evaluate whether the hidden scratchpads contained alignment-faking reasoning.

Our training experiments were run on Anthropic’s infra, and we did not release training code. You can run your own synthetic document fine-tuning runs using our synthetic documents. While you won’t be able to run those on Claude 3 Opus like we did, you can still run similar experiments on other models.

Here are some questions we are interested in and which we did not investigate in our paper:
