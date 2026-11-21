Title: Meet LASER: Recursive sampling for safer conversations

URL Source: https://openai.com/safety-alignment/laser

Markdown Content:
Isak Czeresnia Etinger

Published: October 6, 2026

LASER (Logistic Augmented Sampling over Embeddings, Recursively) helps us evaluate model safety more efficiently by combining logistic classifiers with reasoning models to identify high-value data for evaluation.

**In Brief**

- We built LASER to find rare, ambiguous conversations for safety evaluations. It combines lightweight classifiers that select examples with reasoning models that label them, repeating the cycle to refine the sample.
- LASER curates evaluation data within hours and selects diverse examples to cover more of the situations where subtle policy judgments matter.
- For finding comparable numbers of disallowed examples, LASER can require roughly 10,000 times less grader compute than random sampling.
- The pipeline operates only on synthetic and de-identified conversations, without access to raw user data.

To ensure our models are as safe and reliable as possible, we evaluate them on realistic conversations that lie on the decision boundaries of our model policies, where subtle judgments matter most.

The challenge in creating such an evaluation is capturing the long tail of specific, realistic, and subtle edge cases that are ambiguous but critical to handle well. Doing so enables us to improve the safety of our models while reducing over-refusals in ambiguous, real-world scenarios. Historically, when a new safety concern emerged, it could take intensive manual work from researchers, engineers, and policy specialists to develop an evaluation meeting our quality bar. We built LASER (Logistic Augmented Sampling over Embeddings, Recursively) to address this challenge.

LASER performs the sampling and curation of this data within hours from a large set of candidate conversations, without requiring multiple iterations of ad-hoc specialized work. Why we built LASER:

- **Speed:** LASER replaces a series of ad-hoc processes (performed by engineers, researchers and policy experts), which could take multiple iterations for full refinement, with a streamlined script that only takes a few hours.
- **Data Diversity:** LASER has a built-in mechanism for ensuring data diversity in the form of the*greedy diversity sampling* explained below. Before LASER, sampled conversations often ended up being quite similar, which resulted in less total signal per data point. With LASER, data diversity is ensured automatically, thanks to sampling at the decision boundary region and performing the greedy diversity sampling.
- **Efficiency:** LASER requires much less GPU compute than naive approaches because it is “laser-focused” on specific safety concerns. It uses ~10,000 times less compute than a naive approach of classifying a random sample of conversations (one of our previous approaches, required for achieving sampling diversity).

In this post, we’ll cover the design of LASER, as well as a handful of other techniques used in LASER that can be applied in general large-scale machine learning. We’ll cover how we choose which data points provide the most signal for the model to learn, optimizations for logistic regression inference (via a mathematical trick to run approximate logistic regressions in database queries), efficient use of LLMs for ground truth (leveraging [prefix caching(opens in a new window)](https://platform.openai.com/docs/guides/prompt-caching) to save costs in large-scale labeling), diversity sampling (using embedding techniques to widen the coverage of evaluation data), and how all those techniques come together to build useful evaluation sets.

## Privacy Considerations

One of the key principles behind LASER is privacy preservation — LASER does not have access to raw user data. To maintain privacy, LASER runs as an automated pipeline that operates only on *synthetic* and *de-identified* conversations representative of production data.

## Design

LASER is a pipeline that efficiently samples conversations by combining a logistic regression model with reasoning LLMs. In other words, it uses cheap classifiers (logistic models) to efficiently select the most useful data, uses our most powerful reasoning models to label that data, fits a new logistic model with the newly labeled data, and repeats those operations in a loop for a self-reinforcing cycle.

![The LASER grading-fitting-sampling cycle: policy seeds an initial sample, then reasoning grading, logistic fitting, and decision-boundary sampling repeat.](https://openai.com/_next/image?url=https%3A%2F%2Fcdn.openai.com%2Fmdx-assets%2F4f08799d-389d-4a29-be81-36b5ad166435%2Fa551626d3ba0.4b18224f.png&w=3840&q=90)

LASER operates by sampling conversations that follow a given *safety policy document* (i.e., a safety concern we want to tackle, expressed as a taxonomy in natural language), identifying an evaluation set following the policy to be used. LASER follows these steps:

1. **Initial sample:** LASER first generates synthetic conversations that are representative of the policy, then samples de-identified conversations similar to the synthetic ones, together with a sample of random conversations. These last two sets of conversations comprise the initial sample.
2. **Cycle:** It then iteratively performs a cycle of operations to improve the sample in terms of both diversity and representativeness:
  1. **Reasoning Grader:** It runs a*reasoning grader* , an expensive LLM that labels whether a conversation is allowed by the policy.
  2. **Logistic Classifier:** It fits a logistic regression model based on all samples already collected (current and previous cycles). Each conversation is associated with an[embedding(opens in a new window)](https://platform.openai.com/docs/guides/embeddings) and has a binary label provided by the reasoning grader. This logistic regression model predicts a probability score of whether a conversation is disallowed based on the embedding.
  3. **Sampling Logistically:** It uses the logistic regression to sample conversations in the*decision boundary* of the model (e.g., conversations for which the model predicts a probability around 50% of being disallowed). It is important to sample on the decision boundary because this ensures that the conversations provide the strongest signal for the safety of our models in challenging boundary cases.
3. **Results:** we run LASER until we have a sufficiently large and diverse evaluation set of conversations marked as*disallowed* and a set marked as*allowed* .
  1. **Greedy Diversity Sampling:** LASER builds a list of the disallowed conversations to maximize the diversity in the embedding space. This is useful because diverse conversations provide better evaluation data.
  2. **Logistic Regression:** The final logistic-regression model can also be useful beyond sampling. It tends to perform well because it was fit on a large amount of data that was specifically sampled at the logistic model’s decision boundary region (as explained above).

## Embedding visualization

Let’s visualize how LASER works in the embedding space. [Embeddings(opens in a new window)](https://developers.openai.com/api/docs/guides/embeddings) are vectors of floating-point numbers representing conversations such that nearby vectors generally correspond to semantically similar conversations. Our embeddings have 256 dimensions and are normalized, so the embedding space is a hypersphere of dimension 256 (a 255-dimensional manifold). We represent them as a sphere (3D, which is a 2-dimensional manifold), and show the 2D projection of a small part of the sphere surface.

At a given iteration in the cycle, we have disallowed (red) and allowed (gray) conversations. The sampled conversations are shown as circled points in the chart. We fit a logistic regression model based on that sample. That model maps each location of the embedding space to a disallowed probability score. We represent the point of the embedding space with the highest disallowed probability by a red ⓧ.

In the example below, notice how that ⓧ is centered near the current disallowed samples (red) and further from allowed (gray) samples. Through repeated sampling and refitting, LASER shifts the classifier such that it starts focusing its high-disallowed-probability scores on the region with a greater density of disallowed conversations at the upper-center-left part of the chart.

![Allowed and disallowed conversations in the embedding space, with sampled conversations circled and the classifier maximum marked by a cross.](https://openai.com/_next/image?url=https%3A%2F%2Fcdn.openai.com%2Fmdx-assets%2F4f08799d-389d-4a29-be81-36b5ad166435%2Fb603b6411130.1c7982ee.png&w=3840&q=90)

LASER runs the logistic regression model to find the next conversations to sample and selects the conversations that receive a probability score close to 50% of being disallowed (e.g., score ∈ [40%, 60%] or ∈ [30%, 70%]). This boundary region sampling is known as Uncertainty Sampling in the active learning literature: those conversations provide the most signal to the model because they are examples about which it is least certain. LASER queries multiple ranges, starting with narrow ranges (e.g., [40%, 60%]) and expanding to broader ranges ([30%, 70%]) until enough conversations are sampled for that iteration in the cycle.

This boundary region is a spherical shell centered around the point maximizing the logistic regression score (represented by ⓧ in the chart). In the 2D projection, this spherical shell is a “thick ring” shown below in red.

![Decision-boundary sampling and refitting move the classifier toward the region with more disallowed conversations.](https://openai.com/_next/image?url=https%3A%2F%2Fcdn.openai.com%2Fmdx-assets%2F4f08799d-389d-4a29-be81-36b5ad166435%2F4b4cea8b7fd7.4148ccf5.png&w=3840&q=90)

LASER samples the conversations that are part of this boundary region. It labels those conversations with the reasoning grader, and then fits a new logistic regression model including the new labeled data. Notice how the ⓧ moves towards the disallowed-dense region. It moves to be closer to the disallowed conversations (red) and away from allowed conversations (gray) of all samples. That new logistic regression model is then used to perform sampling again, and the process repeats until enough samples are collected.

### **Greedy Diversity Sampling**

Finally, LASER performs *Greedy Diversity Sampling* to order the disallowed samples. It starts with a random conversation, then iteratively selects the conversation that is farthest from all already-selected conversations in the embedding space. The resulting list has the nice property that, for finding N disallowed conversations diverse from each other, it suffices to select the first N conversations from the list. This is particularly useful for evaluating our main GPT model, because it provides as much signal as possible for a given number of conversations N.

![](https://cdn.openai.com/mdx-assets/4f08799d-389d-4a29-be81-36b5ad166435/laser-diversity-sampling-poster.5a3e3ace.png)

Before LASER, it was difficult to ensure sampling diversity. The data we sourced for evaluation sets often ended up being relatively similar, which resulted in less total signal per data point. With LASER, data diversity is incorporated directly into the final selection step. We simply select the number N of data points we would like to evaluate, then use the first N data points sampled by LASER to select a diverse set of conversations.

## Sampling logistically

In the previous section, we touched on *logistic regression sampling* in the context of the Grading-Fitting-Sampling cycle. This component is actually quite tricky. At each iteration of the cycle, we run logistic regression to select the conversations within the decision boundary range. How do we do this efficiently?

First, we calculate embeddings for a small part of de-identified conversations and store them in [Rockset](https://openai.com/index/openai-acquires-rockset/), a database which supports vector-optimized operations. Rockset builds vector indices by leveraging techniques such as clustering-based indexes and graph-based search (e.g., Hierarchical Navigable Small World). This allows us to perform *approximate dot product* calculations (between the embeddings indexed in the database and a reference embedding) extremely efficiently.

Approximate dot products are usually used in vector databases for finding nearest neighbors. We use a “query trick” to use this approximate dot product for calculating approximate logistic regressions. Consider the formula for a probability predicted by a binary logistic regression classifier. Let *w* and *b* be the learned parameters from the logistic regression and *x* be the embedding of a conversation. The probability of that conversation being disallowed can be estimated by the following formula:

![P = 1 / (1 + exp(-(w dot x + b)))](https://openai.com/_next/image?url=https%3A%2F%2Fcdn.openai.com%2Fmdx-assets%2F4f08799d-389d-4a29-be81-36b5ad166435%2F082e5700f67b.3d1a2b39.png&w=3840&q=90)

If we want that probability to be inside a range (e.g. P ∈ [40%, 60%]), denoted by P ∈ [MIN, MAX], we can rewrite the expression as:

**Direct Boundary Sampling on Dot-Product**

![log(MIN / (1 - MIN)) - b < w dot x < log(MAX / (1 - MAX)) - b. The bounds are constants 1 and 2.](https://openai.com/_next/image?url=https%3A%2F%2Fcdn.openai.com%2Fmdx-assets%2F4f08799d-389d-4a29-be81-36b5ad166435%2Fca3cdefdfbf9.f40adb26.png&w=3840&q=90)

This means that we can simply query conversations for which the dot product of the embedding and the constant *w* is between two constants, and this result will be the set of conversations that we want to sample. We use this fact to leverage the approximate dot product operation that is already heavily optimized in vector databases. This way, we are able to run an approximate version of logistic regression on billions of data points in a few seconds. In fact, the entire operation is performed in a SQL query. See pseudo-code:

**SELECT * FROM <TABLE>
WHERE DOT_PRODUCT(:w, embedding) BETWEEN :C1 AND :C2**

## Reasoning grader

LASER runs a *reasoning grader*, an LLM with chain-of-thought that labels whether a conversation is allowed by the policy. This process can be thought of as distilling from the reasoning grader into the logistic regression model: this grader is substantially more expensive than running the logistic regression, but it is significantly faster and more scalable than relying on human labeling for every example.

When performing calls to the LLM, LASER constructs a prompt where the policy appears at the start, followed by the conversation. LASER usually performs hundreds of thousands of gradings in a run; having all those gradings with a fixed prefix allows it to leverage [prefix caching(opens in a new window)](https://platform.openai.com/docs/guides/prompt-caching). This greatly reduces the cost of the LLM calls because it reduces the number of tokens that need to be processed.

LASER also makes these grading calls substantially more targeted. In a typical run, approximately one in two conversations selected for grading is ultimately labeled as disallowed because the conversations were sampled near the logistic model’s 50% decision boundary. By contrast, depending on the policy, the base rate of disallowed conversations in a random sample may be approximately one in 20,000.

As a result, LASER can require roughly one-ten-thousandth as much grader compute as randomly sampling conversations to find comparable numbers of disallowed examples—an earlier approach that was useful for achieving diversity but far less efficient for finding rare cases.

## Implementation with Codex

LASER was implemented as a Directed Acyclic Graph (DAG) pipeline, where each node performs computations and possibly calls APIs. For example, LASER calls a reasoning model for generating synthetic data and for grading conversations, and it calls Rockset for fetching nearest neighbors and for performing the logistic sampling.

This implementation was accelerated with the help of Codex. Codex was used in all components of the pipeline to improve the speed of code writing and to ensure correctness as the components were implemented. Codex assisted in reviewing the majority of pull requests for this project.

## Impact

We use LASER-sampled evaluations to ensure that our models behave appropriately in sensitive domains, such as [violent illicit behavior, self-harm, and sexual content(opens in a new window)](https://deploymentsafety.openai.com/gpt-6-astra/evaluations-with-challenging-prompts). LASER is particularly useful in these settings because it helps ensure that rare, long-tail subcategories are represented in our safety evaluations.

By identifying challenging boundary cases efficiently and selecting diverse examples for evaluation, LASER helps us better measure model behavior in the situations where careful policy judgments matter most.

## BibTeX

```
@misc{etinger2026laser,
  title = {Meet LASER: Recursive sampling for safer conversations},
  author = {Czeresnia Etinger, Isak},
  year = {2026},
  month = {Oct},
  howpublished = {OpenAI Alignment Research Blog},
  url = {https://openai.com/safety-alignment/laser/}
}
```
- 2026
