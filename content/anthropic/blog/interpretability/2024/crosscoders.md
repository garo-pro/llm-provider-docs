Title: Sparse Crosscoders for Cross-Layer Features and Model Diffing

URL Source: https://transformer-circuits.pub/2024/crosscoders

Markdown Content:
This note introduces sparse crosscoders, a variant of sparse autoencoders (e.g. 

This note will cover some theoretical examples motivating crosscoders, and then present preliminary experiments applying them to cross-layer superposition and model diffing. We also briefly discuss the theory of how crosscoders might simplify circuit analysis, but leave results on this for a future update.

According to the superposition hypothesis, neural networks represent more features than they have neurons by allowing features to be non-orthogonal 

At first blush, the idea that this kind of superposition might be spread across layers might seem strange. But if we think about it carefully, it's actually relatively natural in the context of a transformer with a reasonable number of layers.

One interesting property of transformers is that, because the residual stream is linear, we can draw them as different, equivalent graphs. The following graph highlights the idea that two layers can be thought of as "almost parallel branches", except that they have an extra edge that allows the earlier layer to influence the later.

If we consider a one-step circuit computing a feature, we can imagine implementations where the circuit is split across two layers, but functionally is in parallel. This might actually be quite natural if the model has more layers than the length of the circuit it is trying to compute!

If features are jointly represented by multiple layers, where some of their activity can be understood as being in parallel, it's natural to apply dictionary learning to them jointly. We call this setup a crosscoder, and will return to it in the next section.

It's worth noting that jointly applying dictionary learning to multiple vectors is precisely what we do when models literally have parallel branches with cross-branch superposition 

Crosscoders can help us when there's cross-layer superposition, but they can also help us when a computed feature stays in the residual stream for many layers. Consider the following hypothetical "feature lifecycle" through the residual stream:

If we tried to understand this in terms of a residual stream feature at every layer, we'd have lots of duplicate features across layers. This can lead to circuits which seem much more complex than they need to.

Consider the following hypothetical example, in which features 1 and 2 are present by layer L, and are combined (say via an "and") to form feature 3 via MLPs in layers L+2 and L+3, and then all three features persist in layer L+4. On the left panel of the figure below, we see that per-layer SAEs would produce 13 features in total, corresponding to features 1, 2, and 3 at each layer they are present. The causal graph relating them has many arrows, most for persistence (a feature causes itself in later layers) and two for each of the stages in which feature 3 is computed from 1 and 2. An ideal crosscoder picture, on the right, would have just three features and a simple causal graph.

This means that crosscoders may also give us a strategy for radically simplifying circuits if we use an appropriate architecture where, as in the above picture, feature encoders read in from a single residual stream layer and their decoders write out to downstream layers.

As an example suppose we have feature 

We note, however, that there are some conceptual risks with this approach – the causal description it provides likely differs from that of the underlying model. We plan to explore this approach further in future updates.

Where autoencoders encode and predict activations at a single layer, and transcoders 

We can think of autoencoders and transcoders as special cases of the general family of crosscoders.

The basic setup of a crosscoder is as follows. First, we compute the vector of feature activations 

where 

And have a loss:

Note that the regularization term can be rewritten as:

That is, we weight the L1 regularization penalty by the L1 norm of the per-layer decoder weight norms (

However, there are two reasons to prefer the L1 norm version:

On the other hand, the L2 version more efficiently optimizes the frontier of MSE and global L0 across all layers of the model. Thus, for applications where uncovering layer or model-specific features is not important, and where it is not important to be able to compare loss values to per-layer SAEs, the L2-of-norms version may be preferable. In this report, all experiments used the L1-of-norms version.

This basic version above is what we'd call an "acausal crosscoder". Many variants are in fact possible. In particular, several important dimensions are:

The following table summarizes the variants:

We have found both weakly and strictly causal crosscoders helpful for simplifying feature interaction graphs in our circuits work, but there remain open questions as to how faithfully validate these analyses. Note that strictly causal crosscoder layers as presented here cannot capture the computation performed by attention layers. Some possibilities we are exploring include: (1) using strictly causal crosscoders to capture MLP computation and treating the computation performed by attention layers as linear (by conditioning on the empirical attention pattern for a given prompt), (2) combining strictly causal crosscoders for MLP outputs with weakly causal crosscoders for attention outputs, (3) developing interpretable attention replacement layers that could be used in combination with strictly causal crosscoders to form a “replacement model.”

Can crosscoders actually uncover cross-layer structure? To explore this question, we first trained a global, acausal crosscoder on the residual stream activations of all layers of an 18-layer model. We compared its performance to that of 18 SAEs trained separately on each of the residual stream layers. We used a fixed L1 coefficient for the sparsity penalty. Note that we designed our loss to be comparable to a baseline SAE loss with the same L1 penalty, as discussed above. We separately normalize the activations of each layer prior to training the crosscoder, so that each layer contributes comparably to the loss.

For each approach, we swept over the number of training steps and the number of total features to select the optimal number of features at different FLOPS budgets. We are interested in how the dictionary performance scales with the total number of features in the crosscoder / across all SAEs, and with the amount of compute used in training. Note that for a model with L layers, a global, acausal crosscoder with F total features uses the same number of training FLOPS as a collection of per-layer SAEs with F features each (and thus with L×F total features). Or viewed another way, a collection of single-layer SAEs with F total dictionary features summed across all the SAEs can be trained with L times fewer FLOPS than a single crosscoder with F dictionary features. Thus, crosscoders must substantially outperform SAE on a “per-feature efficiency” basis to be competitive in terms of FLOPs.

First we measure the eval loss of both approaches (MSE + decoder norm-weighted L1 norm, summed across layers):

We found that, controlling for the total number of features across layers, crosscoders substantially outperform per-layer SAEs on eval loss. This result indicates that there is a significant degree of redundant (linearly correlated) structure across layers, which are interpreted by the crosscoder as cross-layer features. However, with respect to training FLOPS, crosscoders are less efficient than per-layer SAEs at achieving the same eval loss, by a factor of about 2 at large compute budgets.

In other words, for a fixed number of total features, crosscoders are able to make more efficient use of their resources by identifying shared structure across layers, allowing them to lump together identical features across layers as a single cross-layer feature, which frees up the crosscoder’s resources to spend on other features. However, identifying this structure costs compute at training time.

However, eval loss is only one measure of the crosscoder’s usefulness. Since our loss scales the sparsity penalty by the sum of decoder norms across layers, it effectively measures (an L1 relaxation of) the sparsity of (feature, layer) tuples. Thus, it provides a sense of how well any single layer of the model can be described as a sparse sum of crosscoder features, vs. SAE features. However, we may also be interested in how well the activity across the entire model can be described as a sparse sum of crosscoder features, vs. SAE features. For this purpose, the metric of interest is the (MSE, L0) value of each method, where in the per-layer SAE case we sum L0 norm across all the SAEs. We show (MSE, L0) values at optimal values of SAE / crosscoder training loss over a set of values of training FLOPS.

Viewed from this perspective, crosscoders provide a dramatic benefit over per-layer SAEs. By consolidating shared structure across layers, they exhibit a much less redundant (and therefore more concise) decomposition of the entire model’s activations. In theory, the same consolidation might be achievable via post-hoc analysis on SAE features, e.g. by clustering features based on the similarity of their activations. However, in practice, this analysis may be difficult, particularly due to stochasticity in SAE training. Crosscoders effectively “bake in” this clustering at training time.

Summarizing the results at a high level, the efficiency crosscoders and per-layer SAEs can be compared in essentially two ways. In terms of the tradeoff between reconstruction error and the sparsity of the feature set used to reconstruct a single layer’s activity, crosscoders make more efficient use of dictionary features but less efficient use of training FLOPS. In terms of the reconstruction error / sparsity tradeoff in reconstructing the entire model’s activity, crosscoders provide an unambiguous advantage, by resolving redundant structure across layers.

We next conducted some basic analyses of the crosscoder features. We were especially interested in features’ behavior across layers: (1) Do crosscoder features tend to be localized to a few layers, or do they span the whole model? (2) Do crosscoder features’ decoder vector directions remain stable across layers, or can the same feature point in different directions in different layers?

Addressing question (1), below we plot the decoder weight norms of 50 randomly sampled crosscoder features across the layers of the model (which are representative of trends we have observed in the full collection of features). For each feature, we rescale the norms so that the maximum value is 1, for ease of visual comparison.

We see that most features tend to peak in strength in a particular layer, and decay in earlier and later layers. Sometimes the decay is sudden, indicating a localized feature, but often it is more gradual, with many features having substantial norm across most or even all layers.

The ability to track the presence of features across layers is spiritually similar to results by Yun et al. 

Returning to the above plot, is the existence of gradual formation of features distributed across layers evidence for cross-layer superposition? While it's definitely consistent with the hypothesis, it could also have other explanations. For example, a feature could be unambiguously produced at one layer and then amplified at the next layer. More research – ideally circuit analysis – would be needed to confidently interpret the meaning of gradual feature formation.

We now return to the second of our original questions, regarding the embedding directions of crosscoder features. Below, for a few example crosscoder features, we show:

The leftmost column is an example of a feature whose decoder direction drifts across layers at roughly the same spatial scale at which its norm decays. The middle column is an example of a feature whose decoder direction is fairly stable over the layers in which the feature has appreciable norm. The right column is an example of a feature that persists throughout the model, but with rapidly changing decoder direction.

These examples were selected to illustrate the range of feature archetypes we find. Below, we show this information for 36 randomly selected features to give a more representative picture.

Overall, we find that most features’ decoder directions are much more stable across layers than would be expected by chance, but also that they drift substantially across layers, even in layers where the feature decoder norm remains strong. The specific behavior varies considerably by feature. This suggests that the cross-layer features uncovered by our crosscoders are not simply passively relayed via residual connections.

Note that we do not conduct a systematic analysis of qualitative feature interpretability in this work. Anecdotally, we find that crosscoder features are similarly interpretable to sparse autoencoder features, and crosscoder features that peak in a particular layer are qualitatively similar to features obtained from a sparse autoencoder trained on that layer. We plan to more rigorously evaluate crosscoder feature interpretability in future work.

We experimented with locally masked “convolutional” variants of crosscoders, in which each feature is assigned a local window of K layers that it is responsible for encoding / decoding. We hoped that this would allow us to capture the benefits of crosscoders while minimizing the FLOPS expense at crosscoder training time. However, we found that eval loss interpolated fairly linearly as we varied the convolutional window K from 1 (per-layer SAE case) to n_layers (global acausal crosscoder) – there was no obvious inflection point that optimized the performance / cost tradeoff. Put another way, the performance of a locally masked cross-coder was similar to that of a smaller, FLOPS-matched global crosscoder. This is consistent with the picture from the distribution of features across layers, as seen in the previous section.

We also experimented with “weakly causal” crosscoders.  We focused in particular on an architecture in which each feature is assigned an encoder layer i – its encoder reads in from layer i alone, and its decoder attempts to reconstruct layer i and all subsequent layers.  We found that in terms of eval loss performance, this architecture’s FLOPS efficiency is in between that of per-layer SAEs (slightly worse) and global, acausal crosscoders (slightly better).  With respect to dictionary size, its performance lagged behind that of global crosscoders by a factor of 3 to 4.

We have also conducted preliminary experiments with strictly causal “cross-layer transcoders,” in which each feature reads in from the residual stream at a layer L, and attempts to predict the output of the MLPs in layers L, L+1, L+2, … NUM_LAYERS. When examining the decoder norms of these features, we find a mix of:

One interesting application of crosscoders is to analyze the differences in feature sets before and after an MLP layer, and the computations that give rise to “new” features in the MLP output (see the section on Model Diffing for related experiments analyzing cross-model differences). To achieve this, we can train a crosscoder on the pre-MLP residual stream space and the outputs that the MLP writes back to the residual stream. We use a masking strategy in which the features’ encoders read only from the pre-MLP space, but their decoders attempt to reconstruct both the pre-MLP activity and the MLP output. Note that this architecture differs from a regular transcoder, in which the features are only responsible for reconstructing the MLP output.

This architecture has two nice properties. First, it allows us to identify features that are shared between the pre and post-MLP spaces, and features that are specific to one or the other. To see this, we can plot the relative norms of the decoder vectors within each space. Remarkably, we see a clear trimodal structure, corresponding to pre-only, shared, and post-only (i.e. “newly computed”) features.

Second, because we constrained the encoder vectors to live only in the pre-MLP space, this architecture allows us to analyze how these “newly computed” features were computed from existing features in the residual stream (similar to [how one would analyze a transcoder](https://www.lesswrong.com/posts/YmkjnWtZGLbHRbzrP/transcoders-enable-fine-grained-interpretable-circuit)). In particular, the inputs to a newly created feature can be computed by taking the dot product of the downstream feature’s encoder vector with the upstream features’ decoder vectors, and weighting by the source features activation (either in a particular context, or averaged over dataset examples).  Anecdotally, we often find that the post-MLP feature represents a more abstract concept and its strongest inputs are specific instances of that concept.  For instance, we find one post-MLP feature that activates on words indicating uniqueness, like “special,” “particular,” “exceptional,” etc., and its pre-MLP inputs each fire for particular words in this category, in particular contexts.

We also analyzed the extent to which “stable” features (arbitrarily those with between 0.3 and 0.7 relative decoder norm weight in post-MLP space) tend to be embedded along similar directions in pre- and post-MLP space. Interestingly, we found a positive correlation on average, but with high variance, and relatively low in absolute terms. This result may explain why feature directions tend to drift across layers – it appears that MLP layers relay many features without nonlinear modification, but along a different axis .

We introduced crosscoders as a way to understand cross-layer features, but the same approach can be used to extract cross-model features. In this section, we'll study the use of cross-model features to compare and "diff" models. Our results here are very preliminary, and while there are significant signs of life, we also find that this strategy produces many features we don't understand.

There's a long history of researchers seeking to compare neural networks. Of course, comparing the functional differences of different neural networks – for example, measuring their performance on benchmarks – is a central part of the machine learning paradigm. But it's natural to ask deeper questions. How do their representations compare? How do they compare mechanistically? A host of methods have developed attempting to address these questions. We can divide them into a few categories:

Entire Representations. A significant body of work studies how similar two neural network representations are, often producing aggregate measures of representation similarity. The earliest attack on this problem we're aware of is a philosophical paper by Laakso & Cottrell 

Neurons and Features. If our goal is interpretability, we likely want a finer grained way to reason about the similarity of neural networks. We want to know if neurons or features are similar, even if the networks as a whole are not. We also wish to know what those similar or dissimilar features actually are, and to what extent there may be "universal" features across models. Early work by Li et al. 

Other Interpretability Objects. Although the existence of universal features and circuits is the most investigated topic in this space, it's worth noting the existence of preliminary evidence that other "interpretability objects" may be universal. Once analogous features are discovered between models, it may be possible to identify analogous circuits. Schubert et al. 

Model Diffing. As the idea of universal features and circuits became more widespread, interest began to arise in the idea of "diffing" models as a way to make safety auditing easier. Just as we review software in terms of incremental diffs, you might hope to review the safety of models by focusing on how it has changed from a previously deployed model. To the best of our knowledge, this "model diffing" problem was originally articulated by the OpenAI Clarity Team in 2018.[write up](https://www.alignmentforum.org/posts/X2i9dQQK3gETCyqh2/chris-olah-s-views-on-agi-safety) on the agenda of the clarity team (Gabriel Goh, Nick Cammarata, Chelsea Voss, Ludwig Schubert, Michael Petrov, Shan Carter, and Chris Olah). It was also pitched as an important safety strategy as part of Chris Olah's talk at the Iceland safety workshop in 2019.

Comparison of Finetuned Models. An important application of model diffing is comparing multiple finetuned versions of one model, or comparing a finetuned model to the original version before finetuning. This is both of immediate applied interest (finetuning is used extensively for commercially deployed models and it would be useful to compare different finetuning strategies) and of longer-term safety interest (many theoretical arguments for safety risk suggest that finetuned models are more likely to be dangerous, especially if they're finetuned with RL).

Several recent results suggest that finetuned models use similar mechanism as the base model from which they were trained 

One of the exciting things about crosscoders is they're not limited to closely related models (such as a finetuned version of a particular base model). Instead, we can get cross-model features between any models, including across:

This isn't limited to two models. Subject to computational constraints, we can get cross-model features between arbitrary numbers of models. Of course, when models are significantly different, we may not know what the analogous layers to compare between the models are; in this case, we may wish to also have our cross-coder extend across layers.

Once we get cross-model features, we can study:

For this preliminary writeup, we will limit ourselves to two experiments. First, we will study how finetuning affected Claude 3.0 Sonnet, diffing the middle layer of the base model against its finetuned counterpart. We'll then turn our attention to scaling, and study how features and their distribution over layers varies as a function of scale. These are very preliminary experiments, and we'll only provide initial analyses, leaving more detailed investigations to potential future work.

We trained a crosscoder with 1 million features on the residual stream activations from the middle layer of Claude 3 Sonnet and the base model from which it was finetuned. We wanted to test whether the crosscoder could decompose the models’ activations into shared features and model-specific features. These model-specific features would indicate features learned, or forgotten, during finetuning.

To test this, we looked at the relative norms of feature decoder weights in the two models. Remarkably, we found that features cluster into three obvious groups – base model-specific features, finetuned model-specific features, and shared features. In this example, there are between four and five thousand model-specific features for each model, out of a total 1 million features.

We found a few particular examples of finetuned model-specific features particularly notable.

We also noticed some interesting base model-specific features, which represent Human/Assistant interactions that are at odds with the kinds of interactions Claude is trained to have:

These features are cherrypicked. Unfortunately, we've also found that the majority of the model-exclusive features are not immediately interpretable. However, inspecting the features that activate on tokens of interest often reveals clearly interpretable features, leading to a mixed picture we're still working to understand.

For the shared features, we checked whether their decoder vectors are aligned in the two models. In almost all cases, they were highly aligned, suggesting that these features do in fact represent the same concept, and perform the same function, in the two models. However, we also found that for a few thousand features, the correlation was very low or even negative. We have not investigated this phenomenon in depth, but we suspect that these indicate cases where the finetuned model uses a concept that was present in the base model, but in a new way.

Note that recent work by Kissane et al. 

We trained an acausal crosscoder on ten evenly spaced layers from three models of increasing sizes. We were interested in how much shared structure exists across different models, and which layers correspond to which across models. Our analysis is in very preliminary stages; however, we have obtained one interesting result.

For each feature, we measured the norm of its decoder in each (layer, model) pair (producing a 3 models × 10 layers = 30-dimensional vector for each feature). We then applied nonnegative matrix factorization to these norm vectors across all features. The NMF components provide a window into which (layer, model) pairs tend to share features, and also which features contribute to this shared structure.

The example below shows the results of running NMF with four components, assigning a different color to each component (left), and the spectrum of feature loadings onto each component (right). Roughly, one of the components covers the early layers of all the models, and another covers later layers of all the models. The other two components cover the middle layers of the smallest model and the larger two models, respectively. This suggests that qualitatively new representations emerge in middle model layers as model scale increases. We are interested in qualitatively exploring the features responsible for these differences in future work.

One of the exciting things about crosscoders (and especially the model diffing component) is that they may give us a fresh line of attack on many very basic questions. To give just a few examples:

It would be very exciting to see some of these investigated. However, this does rely heavily on crosscoder-based model diffing, and as discussed above, our results there are a bit mixed.

One might see the transition from interpreting neurons, to SAE or transcoder features, to crosscoders as progressively moving away from the literal, surface-level model that we're studying. Each step of abstraction allows us to escape unfortunate details of how circuits may be scaffolded onto models, but at the cost of moving further away from the ground truth.

If we set things up carefully – in particular, using "error features" (see eg. 

Despite these downsides, it seems quite plausible that the interpretability advantages of studying simplified isomorphisms to the underlying model will be significant enough to favor this approach.

We are deeply grateful to our colleagues Adam Jermyn, Brian Chen, Craig Citro, Emmanuel Ameisen, Thomas Henighan, Kelley Rivoire, Nick Turner, Adam Pearce, Rodrigo Luger, Shan Carter, Siddharth Mishra-Sharma, Hoagy Cunningham, Andrew Persic, Callum McDougall, Trenton Bricken, and Wes Gurnee. Thanks as well to Martin Wattenberg, Daniel Murfet, Neel Nanda, Liv Gorton, Gabriel Goh, and Nick Cammarata for helpful remarks.
