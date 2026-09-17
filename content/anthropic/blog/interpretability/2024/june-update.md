Title: Circuits Updates - June 2024

URL Source: https://transformer-circuits.pub/2024/june-update

Markdown Content:
We report a number of developing ideas on the Anthropic interpretability team, which might be of interest to researchers working actively in this space. Some of these are emerging strands of research where we expect to publish more on in the coming months. Others are minor points we wish to share, since we're unlikely to ever write a paper about them.

We'd ask you to treat these results like those of a colleague sharing some thoughts or preliminary experiments for a few minutes at a lab meeting, rather than a mature paper.

There have been a few different papers in recent months proposing alternative approaches to training SAEs. These works point to big wins being available in the Average Number of Active Features/Mean Squared Error (henceforth L0 and MSE) tradeoff space, primarily due to alleviating the shrinkage issue in which the L1 penalty makes the optimal-loss level of feature activation lower than the ‘true’ activation leading to less accurate reconstruction for a given level of sparsity.

These papers include:

There was also a post in our February Update – [Using a Tanh penalty.](https://transformer-circuits.pub/2024/feb-update/index.html#dict-learning-tanh) (Jermyn et al) – which found that replacing the L1 penalty with a Tanh function improved the L0/MSE tradeoff, but they also found that this resulted in significantly less interpretable features. Rajamanoharan et al compare the Gated Sparse Autoencoders (Gated SAEs) to standard SAE features and find no significant difference in the degree of interpretability, while Gao et al compare measures of interpretability between TopK hyperparameters, but do not compare to standard or Gated SAEs.

We are therefore keen to check that we can replicate these improvements in L0/MSE, and also, especially for the TopK SAEs, that these changes aren’t accompanied by regressions in the degree of interpretability of the found features.

We ran a replication of some of these claims and confirmed that we see a significant benefit from shifting from the standard, L1 penalized SAEs, to either Gated SAEs or TopK SAEs, with the performance of the latter two being similar. We used tied weight initialization between the encoder and decoder for both models, and used the unconstrained decoder norm (see [April 2024 Circuits Update](https://s3-frontend.infra.ant.dev/anthropic-serve/adam/drafts/april-2024-update/index.html)) on the Gated SAE but not the TopK SAE.

Our basic results on L0/MSE are shown in the graph below – note that for TopK the L0 is by definition equal to the value of K.

Notably, the amount of optimization that’s gone into training the TopK and Gated SAEs is lower than that going into the standard SAEs. For example, we haven’t implemented the variant of Ghost Grads that [Gao et al](https://cdn.openai.com/papers/sparse-autoencoders.pdf) used to prevent dead features in their recent paper. Despite this, it’s clear that there’s a very significant improvement in (L0, MSE) here, as previously demonstrated by Gao et al and by Rajamanoharan et al.

We can also see significant differences in the distribution of densities between the different approaches. The TopK runs have more features at the upper end of the density range than the Gated SAE, even when the Gated SAE has a higher L0. Both TopK and Gated SAEs display a greater number of dense features than the standard approach.

| Run Type | L0 | Features > 1e-2 | Features > 1e-3 | 
| TopK - K=50 | 50 | 261 | 8076 | 
| TopK - K=20 | 20 | 54 | 2855 | 
| Gated SAE | 37 | 8 | 4998 | 
| Vanilla SAE | 38 | 13 | 360 | 

We can also look at the average level of feature activation, and here the two TopK runs stand out even more with a greater degree of variation in the average activation, with gated SAEs being intermediate between the TopK and standard runs.

It’s clear that TopK represents a Pareto improvement over the standard SAE on the L0/MSE tradeoff, but ultimately to declare it an improvement over the existing baseline we need to confirm that the quality of the features found are comparable or better.

There are two broad ways to evaluate the SAE – we can evaluate the patterns of when features fire, and then we can try to evaluate the quality of the learned basis and how well this basis allows us to understand the semantics of the vector space.

To evaluate the first we evaluated the degree of monosemanticity of the firing patterns of the features. We first set up a manual blind test of the features, in which the set of top-activating contexts were displayed, alongside the first seven of the activation intervals as displayed in feature visualizations [like in Scaling Monosemanticity.](https://transformer-circuits.pub/2024/scaling-monosemanticity/features/index.html?featureId=1M_3)

The lower buckets were not displayed as both Gated SAEs and TopK SAEs have noticeably different patterns of low activations which would have removed the blinding, and it’s unclear that the monosemanticity of these low activations is an important metric for these features.

Activation Consistency

5: Clear pattern with no deviating examples

4: Clear pattern with one or two deviating examples

3: Clear overall pattern but quite a few examples not fitting that pattern

2: Broad consistent theme but lacking structure

1: No discernible pattern


Complexity

5: Rich feature firing on diverse contexts with an interesting unifying theme, e.g. “feelings of togetherness”

4: Feature relating to high-level semantic structure, e.g. “return statements in code”

3: Moderate complexity, such as a phrase, category, or tracking sentence structure e.g. “website URLs”

2: Single word or token feature but including multiple languages or spelling e.g. “mentions of dog”

1: Single token feature e.g. “ the token ‘(‘”



We manually scored 100 features using this rubric without access to which run the features were from. It seems that there’s no discernible difference between the different runs at this scale of analysis and any difference will be slight.[\[1\]](https://transformer-circuits.pub#ftnt1)

As mentioned above, TopK runs exhibit higher density features than are found in comparable standard SAEs. This isn’t necessarily a problem but they have a higher than average impact on the reconstruction so it’s important to check that they are also interpretable. To this end we also looked specifically at the highest density features in a run with K=20, n_features=256k on an 18 layer model.

Firstly, there’s a lot of variation in the extent of high density features. Some runs with higher values of K had several features in the 20–50% region with no clear explanation and these are more worrying. The regime we care most about however is that with a very large number of features and high sparsity, and with the improvements in L0/MSE we can target lower L0 values than with standard SAEs.

In contrast, in a K=100 run, we see dozens of high density features which activate more than 1 in every 10 tokens and which seem largely uninterpretable, and which would be a worrying sign, but given the positive results for lower K values, mostly just suggests that we should aim for greater levels of sparsity.

It therefore seems that these high density features are a common outcome of shrinkage-reduction methods, and are pathological at high values of L0, but also these methods allow for much lower L0 at comparable levels of MSE error and when using these low L0 runs the features are of at least comparable quality to those found with standard SAEs.

The conclusion is that these high density features don’t seem to be pathological, and this also suggests that previous finding from Anthropic that using Tanh(x) instead of the L1 norm as the sparsity penalty, which also leads to improvements on L0/MSE along with high density features, may not have been as pathological as previously thought, at least with high number of learned features in high sparsity regimes.

We also ran 200-250 features from each of four runs through Clerp, our automatic interpretation system, which generates predicted feature activations based on the model-generated feature explanations and scores each feature by the correlation between the real and predicted feature activations.

We found that the scores for the L1 = 5 Gated SAE and K=20 TopK SAE both score as well as the L1=10 standard SAE which has far worse MSE loss and lower L0. These results are clearly consistent with both Gated SAEs and TopK being a step forward in terms of interpretability, though it may be that by having many fewer lower activations, the tasks are easier while the upper and middle activation ranges are very similar, which is consistent with the manual results above.

More detail on Clerp can be found in [Towards Monosemanticity.](https://transformer-circuits.pub/2023/monosemantic-features)

The above results give us confidence that both Gated SAEs and TopK SAEs are strong alternatives to standard SAEs with little downside risk and the potential to be a meaningful improvement. However, it’s still difficult to know whether the basis found by an SAE is better or worse. For example, an approach which solves or reduces shrinkage may significantly improve the MSE loss while the underlying feature basis remains unchanged. Gao et al try to measure the quality of the basis more directly by finding the quality of the best possible probe for certain binary datasets that use only a direction. This seems a principled approach but it's not clear which kinds of datasets we should expect features to discover clean probes for.

Ultimately SAEs need to be judged on whether they provide additional insight into how the model works - can we use it to debug model issues? For steering? For finding circuits? For understanding the impact of fine-tuning? To improve robustness? It’s clear that the evaluations we have at the moment don’t get to the heart of what we care about and we’re excited to work on that and for future work from others which fills this gap.

[Scaling and Evaluating Sparse Autoencoders](https://arxiv.org/html/2406.04093v1) by Leo Gao et al from OpenAI present an alternative method of training sparse autoencoders in which they use a TopK activation function instead of the ReLU used in the original SAE works. By removing the L1 pressure it removes the issue of shrinkage, which is where the optimal loss is achieved by the feature having a smaller activation value than the true value. It also allows us to set the L0 norm directly which is a much more understandable hyperparameter that gets at the thing we’re interested in much more directly than an L1 penalty, and instead more directly penalizes the sparsity that we are interested in.  They show empirically that on the L0-MSE curve which is commonly used to evaluate SAEs that it’s a clear improvement over the vanilla SAE and very similar to the Gated SAE of [Rajamanoharan et al](https://arxiv.org/abs/2404.16014). In their SAE training they find two key tools for reducing the number of dead features - firstly to initialize the encoder as the transpose of the decoder, and secondly using a variant of the ‘ghost grads’ approach in which features which have not fired for some fixed period of time are encouraged to model the reconstruction error with an auxiliary loss using the highest-activating dead features.

They also show a pair of novel methods of trying to evaluate the models. Firstly they take a number of existing classification datasets and, for each SAE, find the accuracy of the best possible linear probe that can be built where the probe direction is aligned with one of the SAE features. They find that in experiments on GPT-2 the probe loss is at a minimum for SAEs with L0 in the range 64–128, and decreases with the number of features. The changes in probe quality over training are rather modest, and it’s unclear to what extent the probes are the kind of things that one should expect to exist as single features, but the ability to evaluate SAEs on terms that are totally external to the technique is exciting and is complementary to the approach of Makelov et al as discussed below. They also evaluate their SAEs by the sparsity of the effect of ablating the feature on the logits, here finding that the greater degree of sparsity is found in the L0 range of 128–256.

See the section in this month’s update on comparing TopK, Gated SAEs with standard SAEs for more information, which replicates Gao et al’s claim that the TopK SAEs are a step forward in the L0/MSE tradeoff. We are glad to see other frontier AI labs sharing the goal of deeply understanding the mechanisms underlying the capability of their models and are grateful for their contributions to this line of research.

As we've [previously discussed](https://transformer-circuits.pub/2024/scaling-monosemanticity/index.html#discussion-limitations), one major obstacle in the development of sparse autoencoders is the lack of a clear “ground truth” objective. The metrics we optimize are only proxies for interpretability. For example, we abandoned our attempts to train autoencoders with a [tanh penalty](https://transformer-circuits.pub/2024/feb-update/index.html#dict-learning-tanh) after finding that the resulting features were harder to interpret, even though they improved L0 and loss recovered. We are therefore excited for the development of other methods for evaluating SAEs.

In [RAVEL: Evaluating Interpretability Methods on Disentangling Language Model Representations](https://arxiv.org/abs/2402.17700), Huang et al. propose a benchmark that tests an interpretability method's ability to isolate attributes and causally intervene on them.

For example: Paris is in Europe, and people in Paris speak French. A good interpretability method should be able to disentangle the model’s representation of these two aspects of Paris. To evaluate this, one can obtain the model’s representation of a similar prompt featuring, say, Tokyo, and then replace only the part of the representation that the method suggests is about the continent. The interpretability method succeeds if, after this activation patching intervention, the model reports that Paris is in Asia, but continues to report that people in Paris speak French.

The authors of RAVEL test several methods on this benchmark and find that the best method is a variation of [Distributed Alignment Search](https://arxiv.org/abs/2303.02536), which learns a linear subspace of a model that has the desired effect when intervened on. Although the authors primarily evaluate RAVEL on supervised methods, we are most excited about using RAVEL as a benchmark for unsupervised methods, as we believe that many behaviors of interest in LLMs may not be easily captured in a dataset that can be used to train disentanglement. One potential limitation of RAVEL is that it may be the case that some concepts cannot be independently manipulated in the model – French, for example, is a romance language whose origins tie it to Europe, and manipulating its location separately from its language may not be entirely possible without damaging the model. While the dependence of the properties may provide an upper bound on the score, relative comparisons between methods on the same model may still be made.

In [Towards Principled Evaluations of Sparse Autoencoders for Interpretability and Control](https://arxiv.org/abs/2405.08366), which outlines many concerns with evaluating SAEs we share, Makelov et al. develop a different evaluation method on the indirect object identification (IOI) task, a popular task in interpretability introduced by [Wang et al., 2023](https://arxiv.org/abs/2211.00593). In this task, the model completes a prompt similar to “When Mary and John went to the store, John gave a book to” with the name “Mary”. The authors list three relevant attributes that each IOI prompt has: the subject (S; John), the indirect object (IO; Mary), and their relative positions (IO-S-S, as opposed to S-IO-S). They then learn supervised dictionaries for the above attributes, which form a “ground truth", as well as two kinds of unsupervised dictionaries, “task SAEs” trained on just the IOI dataset and “full-distribution SAEs” trained on OpenWebText, a general text corpus. Finally, they evaluate these dictionaries with the following criteria:

The first two criteria can be evaluated without reference to the “ground truth” attributes. Only the last metric depends on the specific choice of attributes, as well as additional human judgment in deciding what kinds of predicates are simple. For example, the authors include a predicate about the typical gender of a name in their list, which they find reflected in a few learned features. A priori, any specific language model may not use features corresponding to these specific attributes to solve the IOI task, but the empirical results suggest that the studied model, GPT2-small, does.

The authors verify that the supervised dictionaries perform well according to all three criteria, but the unsupervised dictionaries fall short, with the task SAEs outperforming the full-distribution SAEs. This sets a concrete benchmark that dictionary learning methods can aspire to improve on. They also report on several phenomena:

This work is a step towards evaluating the quality of an overall representation given by an SAE, as opposed to just the quality of an individual feature or its correspondence to a hypothesized probe. Since a model may organize concepts differently than we do, such an ensemble approach to evaluating features is valuable.

[Not All Language Model Features Are Linear](https://arxiv.org/abs/2405.14860) challenges the linear representation hypothesis of LLMs, in which features represent one-dimensional lines (directions) in some latent space, by exploring the existence of multidimensional feature representations. The authors focus on statistically irreducible multidimensional features; e.g., features that cannot be decomposed into statistically independent features or into a mixture of non co-occurring features.

The two most interesting examples of these multidimensional features are the two-dimensional spaces in which days of the week and months of the year are represented; the authors find these features in both Llama 3 8B and Mistral 7B using an unsupervised clustering strategy. In the study, the authors focus on the specific task of modular arithmetic with days and months (e.g., “two days from Monday is…” and “four months from November is…”) to investigate whether these circular representations are utilized for computation, i.e., whether there is a causal relationship between the representation and the output. Through activation patching intervention experiments, the authors demonstrate that circular representations are indeed causally involved in solving these tasks, and that the representation is irreducible: the model internally represents these features using more than one dimension. Specifically, the authors demonstrate that information is encoded in the angle along the day-of-the-week or month-of-the-year circles. This is best demonstrated by their off-distribution interventions, in which they find that the model consistently predicts the correct output token (day or month) regardless of the position of the patched point in the circular subspace (e.g., whether it is inside or outside the unit circle) as long as its angular position is the same.

The authors further isolate the computation to primarily the MLP layers of the model. While attention heads are involved in copying the information needed for the computation to the final token position, intervening on individual attention heads had relatively small effects compared to intervening on the MLP layers.

Interestingly, the study did not find other readily interpretable multidimensional features. While this could be due to limitations of the SAE clustering method used to identify these features, it could also mean these irreducible multidimensional features are rare, and that most concepts can be represented as linear combinations of one-dimensional features.

[\[1\]](https://transformer-circuits.pub#ftnt_ref1) The error bars are probably too confident as the activation consistency measures are quite heavy tailed on the low end and a couple of unintelligible features can have a large impact.
